# 用 Rust 构建命令行待办事项应用 — 第五部分

> [!RECALL]
> 在 part 4 中，你为 `Todo` 手写了 `impl fmt::Display`，让 `println!("{}", todo)` 能输出用户友好的格式。但 `Debug` 只需要一行 `#[derive(Debug)]`。为什么 `Display` 不能用 derive 自动生成？

四个部分走下来，`tasky` 已经有了八个子命令、全局参数、两种测试、模块化的代码结构。part 4 末尾留了五道练习，它们是整门课程里密度最高的实践——每一道都对应一个你在真实项目中迟早会遇到的模式。

本部分不引入大段新知识。我们做两件事：第一，精讲 part 4 的五道练习中**最重要的四道**——`PartialEq`（让你的类型可以被 `assert_eq!` 比较）、`--sort`（闭包与排序）、`From<&str>`（类型转换 trait）、用 `--data-dir` 隔离集成测试（告别 `unsafe` 的 `set_var`）。第二，回答 part 4 结尾那道关于 `retain` 和 `filter` 的思考题，然后对整个课程做一次总结——你学了什么、走了多远、接下来往哪去。

本部分结束时，你的 `tasky` 将拥有可排序的列表、可比较的 `Todo`、干净的测试套件，而你将拿到一张覆盖五个部分所有核心概念的地图。这门课到此完结——后面的路由你自己选。

## 你将构建什么

```
$ tasky add "买菜"
✔ Added #1: 买菜

$ tasky add -p 2 "修复线上 bug"
✔ Added #2: 修复线上 bug   [urgent]

$ tasky add -p 1 "写周报"
✔ Added #3: 写周报   [high]

$ tasky list --sort priority
  Pending (3):
    [2] 修复线上 bug  (2026-08-10 10:05)  !!urgent
    [3] 写周报  (2026-08-10 10:06)  ! high
    [1] 买菜  (2026-08-10 10:04)
```

同时，`tests/cli_test.rs` 将被重写——不再修改 `HOME` 环境变量，不再需要 `unsafe`，每个测试用自己的临时目录，可以并行运行。

## 前置条件

- part 4 的主线内容全部完成：`clear` 命令、`--data-dir` 全局参数、`storage.rs` 单元测试、`Todo` 的 `Display` 实现
- part 3 的练习 1（`tags`）和练习 2（`stats`）已完成

### 为什么是这四道练习

part 4 的五道练习各有侧重，我们挑出四道精讲，标准是**它们在真实 Rust 项目中出现的频率**：

- **练习 1（`PartialEq`）**——几乎所有需要测试的自定义类型都会 derive 它。没有它，`assert_eq!` 无法比较你的类型。
- **练习 2（`--sort`）**——闭包作为参数、`Ordering` 返回值、就地排序，这是你处理集合时的高频操作。
- **练习 3（`From<&str>`）**——类型转换是标准库 API 设计的核心模式，理解 `From`/`Into` 让你读懂大量库代码。
- **练习 5（测试隔离）**——你在 part 3 中用 `set_var("HOME", ...)` 的方式有个根本缺陷：它依赖全局状态。这道练习教你 Rust 测试的正确姿势。

练习 4（改进 `stats` 展示）留给本部分末尾的练习区——它涉及 `chrono` 的时间差计算，值得你自己动手。

## 练习 1：PartialEq——让 Todo 可以被比较

回忆 part 4 中你写的单元测试：

```rust
assert_eq!(loaded[0].id, 1);
assert_eq!(loaded[0].content, "测试任务一");
```

你在**逐字段**比较——逐个字段写断言，啰嗦且容易漏。如果 `Todo` 支持 `==` 运算符，你只需要一行：

```rust
assert_eq!(loaded[0], expected_todo);
```

让类型支持 `==` 的方式是 derive `PartialEq`：

```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Todo {
    // ... 字段不变
}
```

`#[derive(PartialEq)]` 让编译器自动生成一个逐字段比较的实现——等价于手写：

```rust
impl PartialEq for Todo {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
            && self.content == other.content
            && self.completed == other.completed
            && self.created_at == other.created_at
            && self.priority == other.priority
            && self.completed_at == other.completed_at
            && self.tags == other.tags
    }
}
```

前提是每个字段的类型都实现了 `PartialEq`——`u32`、`String`、`bool`、`DateTime<Local>`、`Option<DateTime<Local>>`、`Vec<String>` 全都实现了，所以 derive 可以直接工作。

`assert_eq!` 宏需要两个 trait：`PartialEq`（用来比较）和 `Debug`（用来在失败时打印两个值）。你的 `Todo` 两个都有了。

### 写一个比较测试

在 `todo.rs` 底部添加单元测试模块：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn todo_equality() {
        let ts = Local::now();
        let a = Todo {
            id: 1,
            content: "买菜".to_string(),
            completed: false,
            created_at: ts,
            priority: 0,
            completed_at: None,
            tags: vec![],
        };
        let b = Todo {
            id: 1,
            content: "买菜".to_string(),
            completed: false,
            created_at: ts,
            priority: 0,
            completed_at: None,
            tags: vec![],
        };
        assert_eq!(a, b);
    }
}
```

> [!HEADS-UP]
> 注意 `let ts = Local::now();` 这一行。如果你用 `Todo::new(...)` 创建两个待办来比较，它们的 `created_at` 会**不同**——两次 `Local::now()` 调用之间有微秒级的时间差，`assert_eq!` 会失败。这就是为什么测试里手动构造结构体并共享同一个时间戳。这是一个典型的"测试数据构造"技巧：当你需要两个"相同"的对象时，控制每一个字段，包括时间。

运行：

```bash
cargo test --bin tasky
```

预期输出：三个测试通过（`storage.rs` 的两个加上这个新的）。

> [!ASIDE]
> **`PartialEq` 和 `Eq` 的区别。** 还有一个叫 `Eq` 的 trait，它比 `PartialEq` 更强：要求**每个值都等于自己**（自反性）。标准库中唯一常见的例外是浮点数——`f64` 的 `NaN != NaN`，所以 `f64` 实现了 `PartialEq` 但不能实现 `Eq`。你的 `Todo` 没有浮点字段，加上 `#[derive(PartialEq, Eq)]` 完全没问题。日常写测试用 `PartialEq` 就够了。

## 练习 2：--sort——闭包与排序

part 4 的练习 2 要求给 `list` 命令加排序。这道题的价值在于让你第一次写**返回 `Ordering` 的闭包**——Rust 排序 API 的标准姿势。

### 添加 CLI 参数

在 `cli.rs` 的 `List` 变体中添加 `sort` 字段：

```rust
/// List todos(pending by default)
List {
    /// show all todos including completed
    #[arg(short, long)]
    all: bool,
    /// show above the level's priority
    #[arg(short, long)]
    priority: Option<u8>,
    /// show contain the tag's toso
    #[arg(short, long)]
    tag: Option<String>,
    /// Sort order: id, priority, created
    #[arg(short, long, default_value = "id")]
    sort: String,
},
```

`default_value = "id"` 让用户不传 `--sort` 时默认按 ID（插入顺序）排列。注意这里用的是 `default_value`（字符串）而不是 `default_value_t`——`String` 类型的默认值直接写字符串即可。

### 排序逻辑

`Vec` 的 [`sort_by`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.sort_by) 方法接受一个闭包，闭包接收两个元素的引用，返回 `std::cmp::Ordering`——一个有三个变体的枚举：`Less`（第一个排前面）、`Equal`（顺序不变）、`Greater`（第二个排前面）。

修改 `main.rs` 中的 `cmd_list`：

```rust
fn cmd_list(todos: &[Todo], all: bool, priority: Option<u8>, tag: Option<String>, sort: String) {
    let mut items: Vec<&Todo> = todos
        .iter()
        .filter(|t| all || !t.completed)
        .filter(|t| match priority {
            Some(p) => t.priority >= p,
            None => true,
        })
        .filter(|t| match &tag {
            Some(p) => t.tags.join("").contains(p),
            None => true,
        })
        .collect();

    match sort.as_str() {
        "priority" => items.sort_by(|a, b| b.priority.cmp(&a.priority)),
        "created" => items.sort_by(|a, b| a.created_at.cmp(&b.created_at)),
        _ => {}  // "id"：保持插入顺序，什么都不做
    }

    if items.is_empty() {
        println!("Nothing here. Use `tasky add <text>` to create one.");
        return;
    }

    let label = if all { "All" } else { "Pending" };
    print_todos(label, &items);
}
```

三处变化值得拆解：

**`let mut items`。** 原来的 `items` 是不可变的——`collect()` 之后就再也不改。现在 `sort_by` 要**就地修改** `Vec` 的元素顺序，所以必须加 `mut`。这和 part 4 中 `retain` 需要 `&mut Vec<Todo>` 是同一个道理：凡是就地修改的操作，都需要可变访问。

**`sort.as_str()`。** `sort` 的类型是 `String`，而 `match` 匹配字符串字面量需要 `&str`。`.as_str()` 把 `String` 转为 `&str`——你在 part 3 练习中用过类似的转换（`contains(&p)` 把 `&String` 传给期望 `&str` 的参数）。

**`b.priority.cmp(&a.priority)`。** 数字类型都有 [`cmp`](https://doc.rust-lang.org/std/cmp/trait.Ord.html#method.cmp) 方法，返回 `Ordering`。`a.cmp(b)` 是升序，`b.cmp(a)` 是降序——把两个参数调换位置就反转排序方向。优先级排序用降序（紧急的排前面），创建时间用升序（老的排前面）。

> [!DESIGN-NOTE]
> **为什么用 `String` 而不是枚举？** 更"Rust"的做法是定义一个 `SortOrder` 枚举并让 clap 直接解析它（用 `#[derive(clap::ValueEnum)]`），这样用户输入 `--sort banana` 时 clap 会直接报错，而不是落到 `_` 分支默默按 ID 排序。这里用 `String` 是为了聚焦 `sort_by` 本身。把 `String` 换成枚举是练习 2 的进阶版——本部分末尾的练习区有详细说明。

别忘了更新 `main` 函数中 `List` 分支的调用：

```rust
Commands::List { all, priority, tag, sort } => {
    cmd_list(&todos, all, priority, tag, sort);
}
```

> [!ASIDE]
> `sort_by` 是**稳定排序**——相等的元素保持原来的相对顺序。比如两条待办优先级相同，排序后它们的先后关系和排序前一致。标准库还有 `sort_unstable_by`（更快但不保证稳定性），对几百条待办来说没有区别。

## 练习 3：From<&str>——类型转换 trait

part 4 中你实现了 `Display`，自动获得了 `.to_string()`——那是标准库的 blanket implementation（实现了 `Display` 的类型自动获得 `ToString`）。Rust 还有另一个同样重要的转换模式：`From` 和 `Into`。

[`From<T>`](https://doc.rust-lang.org/std/convert/trait.From.html) trait 定义"如何从 `T` 构造我的类型"。给 `Todo` 实现 `From<&str>`，让用户可以从一个字符串快速创建待办：

```rust
impl From<&str> for Todo {
    fn from(value: &str) -> Self {
        Todo {
            id: 0,
            content: value.to_string(),
            completed: false,
            created_at: Local::now(),
            priority: 0,
            completed_at: None,
            tags: vec![],
        }
    }
}
```

把它放在 `todo.rs` 中 `impl fmt::Display for Todo` 的附近——这两个 `impl` 块都是"为 `Todo` 实现标准库 trait"。

实现 `From` 后你有两种调用方式：

```rust
// 方式一：直接调用 From::from
let todo = Todo::from("买菜");

// 方式二：用 Into（自动获得）
let todo: Todo = "买菜".into();
```

第二种方式来自标准库的 blanket implementation：

```rust
impl<T, U> Into<U> for T where U: From<T> { ... }
```

翻译成人话：**任何类型 `T`，只要 `U` 实现了 `From<T>`，`T` 就自动实现了 `Into<U>`**。你只写了一个 `From` 实现，`Into` 免费附送。这和 `Display` → `ToString` 的模式完全一致。

> [!HEADS-UP]
> **`From` 必须不会失败。** `From` trait 的契约是"完美转换"——不丢失信息、不可能失败。如果转换可能失败（比如从字符串解析数字），用 `TryFrom` 而不是 `From`。你的 `From<&str> for Todo` 永远不会失败——任何字符串都能成为一条待办的内容，所以 `From` 是正确的选择。

在 `todo.rs` 的测试模块中加一个测试验证它：

```rust
#[test]
fn todo_from_str() {
    let todo = Todo::from("买菜");
    assert_eq!(todo.content, "买菜");
    assert_eq!(todo.id, 0);
    assert_eq!(todo.priority, 0);
    assert!(!todo.completed);
    assert!(todo.tags.is_empty());

    // Into 也能用
    let todo2: Todo = "写周报".into();
    assert_eq!(todo2.content, "写周报");
}
```

`From<&str>` 在实际项目中最常见的用途是 API 设计：当你的函数接受 `impl Into<String>` 参数时，调用者既可以传 `String` 也可以传 `&str`——两种都自动转换。这是 Rust 库设计中让 API "好用"的常见技巧。

## 练习 5：用 --data-dir 隔离集成测试

part 3 中你的集成测试长这样：

```rust
fn setup_test_home() -> TempDir {
    let tmp = tempfile::tempdir().unwrap();
    unsafe {
        std::env::set_var("HOME", tmp.path());
    }
    tmp
}
```

这个方案有三个问题：

1. **`unsafe`**——`set_var` 在 edition 2024 中是不安全函数，因为修改环境变量不是线程安全的。
2. **必须串行**——多个测试同时修改 `HOME` 会互相干扰，所以你被迫用 `--test-threads=1`。
3. **依赖全局状态**——测试的正确性取决于环境变量的值，这是一种隐式耦合。

part 4 中你给 `tasky` 加了 `--data-dir` 全局参数。现在用它彻底替代 `HOME` 方案。重写 `tests/cli_test.rs`：

```rust
use assert_cmd::Command;
use predicates::prelude::*;

fn setup_data_dir() -> tempfile::TempDir {
    tempfile::tempdir().unwrap()
}

#[test]
fn test_add_and_list() {
    let tmp = setup_data_dir();
    let dir = tmp.path().to_str().unwrap();

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "add", "测试任务"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Added #1"));

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "list"])
        .assert()
        .success()
        .stdout(predicate::str::contains("测试任务"));
}

#[test]
fn test_done_marks_completed() {
    let tmp = setup_data_dir();
    let dir = tmp.path().to_str().unwrap();

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "add", "完成任务"])
        .assert()
        .success();

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "done", "1"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Done #1"));

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "list"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Nothing here"));
}

#[test]
fn test_remove_nonexistent() {
    let tmp = setup_data_dir();
    let dir = tmp.path().to_str().unwrap();

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "remove", "99"])
        .assert()
        .failure();
}

#[test]
fn test_clear_removes_completed() {
    let tmp = setup_data_dir();
    let dir = tmp.path().to_str().unwrap();

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "add", "将被清理"])
        .assert()
        .success();

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "done", "1"])
        .assert()
        .success();

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "clear"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Cleared 1"));

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "list", "--all"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Nothing here"));
}
```

对比旧版本的变化：`setup_test_home()` 变成了 `setup_data_dir()`——不再设置任何环境变量，只是创建一个临时目录。每个测试通过 `--data-dir` 参数把临时目录传给 `tasky`，数据完全隔离。第四个测试 `test_clear_removes_completed` 是新增的——顺便把 part 4 的 `clear` 命令也纳入了集成测试覆盖。

> [!HEADS-UP]
> **`tmp` 必须活到测试结束。** `tempfile::TempDir` 在被 drop 时自动删除目录。如果你写 `let dir = setup_data_dir().path().to_str().unwrap();`（不绑定到变量），`TempDir` 在这行结束时就被 drop 了——临时目录被删除，后续命令找不到目录而失败。把 `tmp` 绑定为变量，它活到函数末尾，目录在整个测试期间存在。

运行测试——这次**不需要** `--test-threads=1`：

```bash
cargo test
```

四个集成测试可以并行运行，因为每个测试用自己的临时目录，互不干扰。`unsafe` 也从你的代码中彻底消失了。

## 回答 part 4 的思考题：retain vs filter

part 4 结尾问：`Vec::retain()` 和 `.filter().collect()` 都能"只保留符合条件的元素"，区别是什么？

先看它们各自做了什么：

```rust
// retain：就地修改原始 Vec
todos.retain(|t| !t.completed);
// 此后 todos 中只剩未完成的——原始数据被修改了

// filter：创建新集合，原始数据不变
let pending: Vec<&Todo> = todos.iter()
    .filter(|t| !t.completed)
    .collect();
// todos 仍然是完整的，pending 是一个新的视图
```

三个维度的区别：

**内存行为。** `retain` 在原始 `Vec` 上操作——把不符合条件的元素移除，后面的元素向前移动填补缺口，不分配新内存。`filter().collect()` 分配一个全新的 `Vec`，把符合条件的元素复制（或复制引用）进去。对几百条待办来说性能没有可感知的差异，但对百万级数据，`retain` 省掉一次完整分配。

**语义。** `retain` 表达"删除不要的"——是一个**修改操作**，需要 `&mut` 访问。`filter` 表达"选出符合条件的"——是一个**查询操作**，只需要 `&` 只读访问。你的 `cmd_clear` 用 `retain`（真的要删除数据），`cmd_list` 用 `filter`（只是展示一个视图，不修改数据）。

**对原始数据的影响。** 这是最关键的区别。`retain` 之后，原始 `Vec` 变了，被删除的元素永远消失。`filter` 之后，原始 `Vec` 完好无损——你可以随时用不同的过滤条件再过滤一次。part 1 中你学过的设计原则在这里回响：**只读操作不修改数据**。`list`、`search`、`stats` 是只读命令，它们用 `filter` 构建视图；`clear`、`remove` 是修改命令，它们用 `retain` 删除数据。

**什么时候选哪个？** 当你需要修改原始集合（删除元素是目的本身）时用 `retain`；当你需要基于条件构建一个新视图、保留原始数据时用 `filter`。如果你发现自己写 `todos = todos.iter().filter(...).cloned().collect();`——过滤完再赋值回原变量——那大概率应该用 `retain`。

## 课程总结：五个部分，一个工具

从 part 1 的空项目到现在，你构建了一个 8 个子命令、约 550 行代码、带单元测试和集成测试的 CLI 工具。更重要的是，你在这个过程中覆盖了 Rust 入门阶段的几乎所有核心概念。

### 概念地图

| 部分 | 你学到了什么 |
|------|-------------|
| part 1 | 结构体、枚举、`match` 穷举匹配、文件 I/O、JSON 序列化、clap derive、所有权的第一课（`String` vs `&str`） |
| part 2 | trait（`Colorize`）、迭代器链（`iter`/`filter`/`map`/`collect`）、`anyhow` 错误处理、`?` 操作符、`#[serde(default)]` 数据迁移 |
| part 3 | `Option<T>`（`Some`/`None`/`match`/`if let`）、模块系统（`mod`/`pub`/`use`/`crate::`）、集成测试（`assert_cmd`）、`cargo install`、edition 2024 的 `unsafe` 规则 |
| part 4 | 核心概念回顾（所有权/借用/`match`/迭代器）、`Vec::retain`、全局参数（`global = true`）、单元测试（`#[cfg(test)]`/`use super::*`）、手写 trait 实现（`fmt::Display`）、blanket implementation |
| part 5 | `PartialEq` derive、`sort_by` 与 `Ordering`、`From`/`Into` 转换 trait、测试隔离（`--data-dir` 替代环境变量） |

这些概念不是孤立的。你在 part 1 学的 `match`，在 part 3 用来解包 `Option`，在 part 4 用来匹配 `sort` 参数，在 part 5 用来选择排序策略——同一个工具在不同场景中反复出现，直到你不再需要思考它。这就是"肌肉记忆"的形成方式。

### 你现在能做什么

学完这五个部分，你应该能够：

- 用 `cargo new` 开始一个项目，添加依赖，理解 `Cargo.toml` 的结构
- 用结构体和枚举建模数据，用 `match` 处理所有分支——编译器帮你检查遗漏
- 看懂借用检查器的大多数报错（"used after move"、"cannot borrow as mutable"），并知道怎么修
- 用 `Option` 和 `Result` 处理不确定性，用 `?` 传播错误，不再到处 `unwrap()`
- 把代码拆分到多个模块，理解 `pub` 的可见性规则
- 写单元测试和集成测试，知道什么时候用哪种
- 为自定义类型实现标准库 trait（`Display`、`PartialEq`、`From`）

你的代码量——`src/` 下约 485 行加上测试——已经达到了 Rust 入门阶段的一个常见标准：能独立编写 500 行以内的程序，熟练使用 `cargo`，理解编译器错误信息。

### 接下来往哪走

`tasky` 到此完结。从这里出发，入门阶段还有几个方向值得探索：

**类 grep 的命令行搜索工具**——字符串处理、正则表达式、递归目录遍历。它会复用你在这里学到的所有 CLI 技能（clap、anyhow、迭代器），同时引入文本处理这个新维度。这是 Rust 官方教程（The Rust Book 第 12 章）的经典项目，也是从"管理自己的数据"到"处理外部数据"的自然过渡。

**Markdown 或 JSON 解析器**——递归下降解析、枚举状态机、字符串切片。如果你想深入理解"数据是怎么被解析的"，这个方向会给你答案。

**终端游戏（贪吃蛇）**——游戏循环、状态管理、`crossterm` 终端控制。如果你想要即时视觉反馈和更快的迭代节奏。

或者，如果你觉得基础概念还需要更多练习，回到 `tasky` 做本部分的练习题——每一道都是独立的，不需要新的教程。

## 检查点

> [!PREDICT]
> 运行之前想一想：重写后的集成测试不再修改 `HOME` 环境变量。`cargo test` 不加 `--test-threads=1` 能正常运行吗？为什么？

**运行以下命令验证你目前的工作：**

```bash
cargo build
```

预期输出：编译成功，无错误。

```bash
cargo run -- add "买菜"
cargo run -- add -p 2 "修复线上 bug"
cargo run -- add -p 1 "写周报"
```

预期输出：三条 `✔ Added #N` 消息。

```bash
cargo run -- list
```

预期输出（默认按插入顺序）：

```
  Pending (3):
    [N] 买菜  (...)
    [N+1] 修复线上 bug  (...)  !!urgent
    [N+2] 写周报  (...)  ! high
```

```bash
cargo run -- list --sort priority
```

预期输出（紧急的排最前）：

```
  Pending (3):
    [N+1] 修复线上 bug  (...)  !!urgent
    [N+2] 写周报  (...)  ! high
    [N] 买菜  (...)
```

```bash
cargo run -- list --sort created
```

预期输出（最早创建的排最前，和插入顺序一致）。

```bash
cargo run -- list --sort banana
```

预期输出：不报错，按默认的插入顺序排列（落入 `_` 分支）。

```bash
cargo test
```

预期输出：所有测试通过——`storage.rs` 的 2 个单元测试、`todo.rs` 的 2 个单元测试、`cli_test.rs` 的 4 个集成测试。不需要 `--test-threads=1`。

**可能的错误：**

- 如果看到 `cannot borrow 'items' as mutable`（在 `sort_by` 处），你可能忘了给 `items` 加 `mut`——`sort_by` 就地修改 `Vec`，需要可变访问。
- 如果看到 `no method named 'cmp' found for reference '&u8'`，检查闭包参数的写法——`a.priority.cmp(&b.priority)` 中 `cmp` 接受引用参数。
- 如果 `assert_eq!(a, b)` 失败但两个 `Todo` 看起来一样，检查 `created_at`——两次 `Local::now()` 的值不同。用共享的时间戳变量。
- 如果集成测试报 `No such file or directory`，你可能没有把 `TempDir` 绑定到变量——临时目录在创建后立即被 drop 了。

## 接下来

这门课到这里结束。五个部分，一个工具，从空项目到可安装、可测试、可扩展的 CLI 应用。

如果你继续用 `tasky` 管理真实的待办事项，你会发现它真的够用——这就是"构建自己用的工具"作为学习方式的价值：你的练习成果不会在课程结束后被丢弃，它继续运行在你的终端里。

Rust 的学习曲线在入门之后会变得平缓。你已经翻过了最陡的那段——所有权、借用、生命周期不再是需要刻意理解的"概念"，而是你写代码时自然遵守的规则。后面的路，选一个方向走下去就好。

## 练习

1. **改进 `stats` 命令。** 给 `cmd_stats` 添加两个指标：完成率（如 `完成率: 60%`）和最旧未完成待办（创建时间最早且未完成的待办的内容和已等待天数）。用 `chrono` 的 `Local::now().signed_duration_since(t.created_at)` 计算时间差，`.num_days()` 提取天数。这是 part 4 练习 4 的延续。

2. **把 `sort` 从 `String` 换成枚举。** 定义 `enum SortOrder { Id, Priority, Created }`，给它加 `#[derive(clap::ValueEnum, Clone)]`，把 `List` 变体中的 `sort: String` 改为 `sort: SortOrder`。完成后 `tasky list --sort banana` 会让 clap 直接报错而不是静默忽略。提示：`match sort` 直接匹配枚举变体，不再需要 `sort.as_str()`。

3. **把 `storage.rs` 的单元测试也改成用临时目录。** 现在 `save_and_load_roundtrip` 还在写入真实的 `todos.json`。在测试中创建 `tempfile::tempdir()`，把路径传给 `save_todos(&todos, Some(&dir))`——注意 `dir` 需要是 `PathBuf` 类型（用 `tmp.path().to_path_buf()` 转换）。完成后你的所有测试都不再触碰真实数据。

4. **实现 `export` 子命令。** 添加 `tasky export <path>`，把所有待办以纯文本格式写入指定文件——每行一条，用你在 part 4 实现的 `Display` trait（`todo.to_string()`）。这是 `Display` 的第一个实际用途：纯文本导出。用 `fs::write(path, content)` 写文件，错误处理用 `anyhow`。

## 来源

1. [std::cmp::PartialEq documentation](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html) — `PartialEq` trait 的 derive 行为、与 `Eq` 的区别、`assert_eq!` 的要求。
2. [std::vec::Vec::sort_by documentation](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.sort_by) — `sort_by` 的签名、闭包返回 `Ordering`、稳定排序。
3. [std::cmp::Ordering documentation](https://doc.rust-lang.org/std/cmp/enum.Ordering.html) — `Less`/`Equal`/`Greater` 三个变体和 `cmp` 方法。
4. [std::convert::From documentation](https://doc.rust-lang.org/std/convert/trait.From.html) — `From` trait 的契约（不可失败）、`Into` 的 blanket implementation、`TryFrom` 的使用场景。
5. [tempfile documentation](https://docs.rs/tempfile/latest/tempfile/) — `TempDir` 的自动清理行为和生命周期注意事项。
6. [clap ValueEnum documentation](https://docs.rs/clap/latest/clap/trait.ValueEnum.html) — 让枚举直接作为 CLI 参数值的 derive 宏。
