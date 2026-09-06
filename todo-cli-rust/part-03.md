# 用 Rust 构建命令行待办事项应用 — 第三部分

> [!RECALL]
> 在 part 2 中，`cmd_list` 的 `priority` 参数类型是 `Option<u8>`。当用户不传 `--priority` 时，这个值是 `None`；传了就是 `Some(n)`。你在 `filter` 链中用 `match priority { Some(p) => ..., None => true }` 处理了这两种情况。这个 `Option` 到底是什么类型？为什么 Rust 不直接用 `null`？

在前两个部分，你构建了一个能用的 `tasky`：六个子命令、优先级标记、关键词搜索、优雅的错误处理、终端着色。所有代码挤在 `main.rs` 一个文件里——大约 250 行。

这个文件还能再长一些，但你大概已经感觉到了几个问题：每次修改 `Todo` 结构体，你得在一个大文件里上下翻找所有引用它的地方；你想写几个测试验证 `cmd_add` 的行为，但测试代码和命令路由混在一起不知道往哪放；你希望别人能用 `cargo install tasky` 一条命令安装你的工具，但不确定该怎么准备。

本部分解决这三个问题。你会给 `Todo` 加上 `completed_at` 字段记录完成时间（深入理解 `Option<T>` 类型），把 `main.rs` 拆分成四个各司其职的文件（掌握 Rust 模块系统），写几个集成测试验证 CLI 行为（第一次接触 Rust 的测试框架），最后用 `cargo install` 把 `tasky` 安装为全局命令。

本部分结束时，你的 `tasky` 将拥有清晰的项目结构、完成时间记录、基本测试覆盖，而且可以用一条命令安装到任何 Rust 开发者的机器上。

## 你将构建什么

```
$ tasky add "准备周报"
✓ Added #1: 准备周报

$ tasky done 1
✓ Done #1: 准备周报

$ tasky list --all
  All (1):
    [1] 准备周报  ✓ done (2026-06-14 17:30)  (2026-06-14 09:00)
```

完成时间 `(2026-06-14 17:30)` 紧跟在 `✓ done` 之后，创建时间 `(2026-06-14 09:00)` 在最后。如果你撤销完成（`tasky done --undo 1`），完成时间自动消失。

项目结构从单文件变为：

```
src/
  main.rs      ← 命令路由（~30 行）
  todo.rs      ← 数据模型和打印（~50 行）
  storage.rs   ← 文件读写（~20 行）
  cli.rs       ← CLI 定义（~50 行）
tests/
  cli_test.rs  ← 集成测试
```

## 前置条件

- part 1 和 part 2 完成的代码（能通过 part 2 检查点的 `tasky` 项目）
- 如果你做了 part 2 的练习 4（`completed_at`）或练习 5（模块拆分），先把那些改动**还原**——本部分会从头实现它们

在 `Cargo.toml` 中添加一个开发依赖项（后面写测试时用）：

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = "0.4"
dirs = "5"
colored = "2"
anyhow = "1"

[dev-dependencies]
assert_cmd = "2"
predicates = "3"
```

`assert_cmd` 和 `predicates` 是 Rust CLI 测试的标准组合——后面会详细解释。`[dev-dependencies]` 中的 crate 只在 `cargo test` 时编译，不会出现在你发布的二进制文件中。

## Option<T>：值可能不存在

在第一和第二部分，你用 `bool` 字段 `completed` 表示待办是否完成。但有一个信息你一直没有记录：**什么时候完成的**。当你回顾一周的任务时，"已完成"只告诉你结果，"2026-06-14 17:30 完成"才告诉你故事。

这个问题引出本部分的第一个新类型：`Option<T>`。

### 为什么需要 Option

`content: String` 和 `id: u32` 总有值——每个待办都有内容和 ID。但 `completed_at` 不一样：刚创建的待办没有完成时间（它还没完成），只有标记完成时才有。

其他语言用 `null` 表示"值不存在"。Rust 没有 `null`。取而代之的是 `Option<T>` 枚举：

```rust
enum Option<T> {
    Some(T),  // 有值，值在里面
    None,     // 没有值
}
```

两个变体：`Some(T)` 表示有值，`None` 表示没有。`<T>` 是泛型参数，意味着 `Option` 可以装任何类型——`Option<String>`、`Option<u32>`、`Option<DateTime<Local>>` 都可以。

你其实已经在 part 2 中用过 `Option` 了——`cmd_list` 的 `priority: Option<u8>` 参数。当时你可能没仔细想它的含义，现在来深入理解。

关键概念：编译器把 `Option<T>` 和 `T` 视为**完全不同的类型**。你不能把 `Option<String>` 当 `String` 用，就像不能把 `String` 当 `u32` 用一样。你必须显式地从 `Option` 中"取出"值，才能操作里面的 `T`。这就是 Rust 消灭 null 指针异常的方式——不存在"你以为有值但其实没有"的运行时崩溃，编译器在编译时就逼着你处理"不存在"的情况。

> [!ASIDE]
> **`Option<T>` 和 `Result<T, E>` 的平行关系。** `Result` 有两个变体：`Ok(T)` 和 `Err(E)`，表达"可能成功也可能失败"。`Option` 有两个变体：`Some(T)` 和 `None`，表达"可能有值也可能没有"。两者的设计哲学完全相同：用一个枚举容器把"异常情况"编码进类型系统，让编译器强制你处理。你已经在 part 2 中用 `Result` 和 `?` 处理了错误，现在用 `Option` 和 `match` 处理缺失值——模式是一样的。

### 用 match 解包 Option

处理 `Option<T>` 最直接的方式是 `match`——和处理 `Commands` 枚举完全相同的模式：

```rust
let completed_info = match todo.completed_at {
    Some(time) => format!("  completed: {}", time),
    None => String::new(),
};
```

当 `completed_at` 是 `Some(time)` 时，`time` 绑定到内部的 `String`，你可以用它构建显示文本。当是 `None` 时，返回空字符串——什么都不显示。

`match` 对 `Option` 的处理和对 `Commands` 的处理没有本质区别——穷举匹配每个变体，编译器确保你没遗漏。这就是你在 part 1 中遇到的穷举检查的又一个应用场景。

### if let：只关心一种情况时的简写

当你只关心 `Some` 而忽略 `None` 时，`match` 需要写一个什么都不做的 `_` 分支，有点啰嗦：

```rust
// 用 match 处理 Option：只有一个分支有实际逻辑
match todo.completed_at {
    Some(time) => println!("completed at {}", time),
    _ => (),  // 什么都不做——啰嗦
}

// 用 if let：同样的逻辑，更简洁
if let Some(time) = todo.completed_at {
    println!("completed at {}", time);
}
```

`if let` 是 `match` 的语法糖——当你只匹配**一个**模式时用它。如果匹配成功，`time` 绑定到内部值，代码块执行；如果不匹配（`None`），整个块被跳过。

`if let` 也可以带 `else`，相当于 `match` 的通配分支：

```rust
if let Some(time) = todo.completed_at {
    println!("completed at {}", time);
} else {
    println!("not yet completed");
}
```

> [!DESIGN-NOTE]
> **什么时候用 `match`，什么时候用 `if let`？** 一个实用的判断标准：如果你需要对每个变体做不同的事（`Some` 显示时间，`None` 显示"未完成"），用 `match`——它让所有情况的处理方式一目了然。如果你只关心一种情况（"有值就打印，没值就跳过"），用 `if let`——它省掉了空分支的噪音。

### if let 也能用于 Result

`if let` 不限于 `Option`——它能匹配**任何**枚举变体。前面 §"为什么需要 Option" 的 ASIDE 已经讲过 `Option` 和 `Result` 的平行关系：`Option` 有 `Some`/`None`，`Result` 有 `Ok`/`Err`。所以 `if let` 对 `Result` 的用法完全一样：

```rust
// 用 match 处理 Result（两个分支都有逻辑）
match load_todos() {
    Ok(todos) => println!("loaded {} todos", todos.len()),
    Err(e) => eprintln!("failed: {}", e),
}

// 用 if let 只关心 Ok（忽略错误——慎用！）
if let Ok(todos) = load_todos() {
    println!("loaded {} todos", todos.len());
}

// 用 if let 只关心 Err（"失败了就警告，但不崩溃"）
if let Err(e) = save_todos(&todos) {
    eprintln!("Warning: failed to save — {}", e);
}
```

最后一个例子非常实用。在 `tasky` 的 `main` 函数中，你用 `save_todos(&todos)?` 把错误直接传播出去让程序退出。但如果你希望"保存失败时打印警告、程序继续运行"，`if let Err` 比 `match` 更简洁——它只有一个分支有实际逻辑，不需要写 `_ => ()` 的空分支。

和 `Option` 的选择标准一样：两个变体都需要处理时用 `match`，只关心一个时用 `if let`。区别在于 `Result` 的场景里，"忽略 `Err`"通常不是好主意（错误应该被处理），所以你会看到 `Result` 更多用 `match` 或 `?`。`if let` 在 `Result` 上最典型的用法就是 `if let Err`——"只关心错误"。

### Option 的常用方法

除了 `match` 和 `if let`，`Option<T>` 还提供了很多方法避免冗长的模式匹配。你在 part 1 中已经见过 `.unwrap_or(0)`（`cmd_add` 计算新 ID 时），这里再介绍几个最常用的：

```rust
let x: Option<String> = Some("hello".to_string());
let y: Option<String> = None;

// .unwrap_or(default) —— 有值返回值，没值返回默认值
x.unwrap_or("default".to_string())  // "hello"
y.unwrap_or("default".to_string())  // "default"

// .is_some() / .is_none() —— 检查是否有值
x.is_some()    // true
y.is_none()    // true

// .map(f) —— 如果有值，对值应用函数 f，返回新的 Option
let len = x.map(|s| s.len());  // Some(5)
let len2 = y.map(|s| s.len()); // None

// .unwrap() —— 强制取值，None 时 panic（谨慎使用！）
x.unwrap()     // "hello"
// y.unwrap()  // panic! 程序崩溃
```

`.map()` 值得多看一眼。它让你在**不离开 `Option` 上下文**的情况下转换内部的值：`Some(5)` 经过 `.map(|n| n * 2)` 变成 `Some(10)`，`None` 经过同样的 `.map()` 还是 `None`。这个模式在链式操作中特别有用——和你在 part 2 中对迭代器用的 `.map()` 是同一个思路。

`.unwrap()` 是 `.unwrap_or()` 的危险版本：它在 `None` 时直接 panic。只有在**你确定** `Option` 一定是 `Some` 时才用它——比如在测试中。在生产代码中，优先使用 `match`、`if let` 或 `.unwrap_or()`。

> [!ASIDE]
> **`Option<T>` 方法速查。** 完整的 `Option<T>` 方法列表在[标准库文档](https://doc.rust-lang.org/std/option/enum.Option.html)中，有几十个。但日常开发中最常用的就这几个：`is_some()`、`is_none()`、`unwrap_or(default)`、`map(f)`、`and_then(f)`。`and_then` 和 `map` 类似，区别是 `and_then` 的闭包返回 `Option<U>`（不会嵌套成 `Option<Option<U>>`），而 `map` 的闭包返回 `U`。你暂时不需要记住所有方法——遇到具体场景时查文档即可。

## 记录完成时间

理论讲完了，来给 `Todo` 加上 `completed_at` 字段。

在 `Todo` 结构体中添加 `completed_at`：

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Todo {
    id: u32,
    content: String,
    completed: bool,
    created_at: DateTime<Local>,
    #[serde(default)]
    priority: u8,
    #[serde(default)]
    completed_at: Option<String>,
}
```

`Option<String>` 让 `completed_at` 可以"不存在"。`#[serde(default)]` 的作用你在 part 2 中已经学过——旧的 JSON 文件没有 `completed_at` 字段，反序列化时自动填充为 `None`。

> [!ASIDE]
> **为什么是 `Option<String>` 而不是 `Option<DateTime<Local>>`？** 你的 `created_at` 已经用了 `DateTime<Local>`（part 2 练习 3），这里却用 `String`，看起来不一致。原因是实用主义：`String` 简单、序列化免费、和 part 1 中 `created_at` 最初的设计一脉相承。如果你想保持一致，完全可以用 `Option<DateTime<Local>>`——只需把 `String` 替换为 `DateTime<Local>`，`chrono` 的 serde feature 会自动处理序列化。本教程选择 `String` 是为了降低这一步的认知负担。

更新构造函数——新创建的待办没有完成时间：

```rust
impl Todo {
    fn new(id: u32, content: String, priority: u8) -> Self {
        Todo {
            id,
            content,
            completed: false,
            created_at: Local::now(),
            priority,
            completed_at: None,  // 新建时没有完成时间
        }
    }
}
```

`None` 不需要类型注解——编译器从字段声明 `completed_at: Option<String>` 推断出这里的 `None` 是 `Option<String>::None`。

现在更新 `cmd_done`，在标记完成时记录时间，撤销完成时清除时间：

```rust
fn cmd_done(todos: &mut Vec<Todo>, id: u32, undo: bool) {
    match todos.iter_mut().find(|t| t.id == id) {
        Some(todo) => {
            let sign = if undo { "Undo" } else { "Done" };
            todo.completed = !undo;
            todo.completed_at = if undo {
                None
            } else {
                Some(Local::now().format("%Y-%m-%d %H:%M").to_string())
            };
            println!("{}{} #{}: {}", "✓".green(), sign, id, todo.content);
        }
        None => {
            eprintln!("Todo #{} not found.", id);
            std::process::exit(1);
        }
    }
}
```

新增的代码只有三行，但信息密度很高。让我拆解一下：

```rust
todo.completed_at = if undo {
    None
} else {
    Some(Local::now().format("%Y-%m-%d %H:%M").to_string())
};
```

这是一个 `if-else` **表达式**——Rust 的 `if-else` 是表达式，有返回值。当 `undo` 为 `true`（用户要撤销完成），`completed_at` 设为 `None`（清除完成时间）。当 `undo` 为 `false`（标记完成），`completed_at` 设为 `Some(时间字符串)`。

`Local::now().format("%Y-%m-%d %H:%M").to_string()` 生成形如 `"2026-06-14 17:30"` 的字符串——和你构造函数中 `created_at` 的格式完全一致。然后用 `Some(...)` 包装成 `Option<String>`，赋给 `completed_at`。

最后，更新 `print_todos` 显示完成时间。找到已完成条目的 `println!`，修改为：

```rust
if t.completed {
    let completed_time = match &t.completed_at {
        Some(time) => format!("  ({})", time),
        None => String::new(),
    };
    println!(
        "    {} {}  {}{}  ({}){}",
        format!("[{}]", t.id).dimmed(),
        t.content.strikethrough(),
        "✓ done".green(),
        completed_time,
        t.created_at.format("%Y-%m-%d %H:%M"),
        priority_tag,
    );
} else {
    println!(
        "    [{}] {}  ({}){}",
        t.id,
        t.content,
        t.created_at.format("%Y-%m-%d %H:%M"),
        priority_tag,
    );
}
```

这里用 `match` 而不是 `if let`，因为两种情况都需要处理：有完成时间就显示，没有就显示空字符串。`match` 让这两种情况对称地排列在一起，一目了然。

注意 `&t.completed_at`——你借用引用而不是移动值。`completed_at` 的类型是 `Option<String>`，`String` 不是 `Copy` 类型，如果写 `match t.completed_at`（不带 `&`），`match` 会**移动** `completed_at` 的值——而 `t` 还在被 `println!` 使用，借用检查器会报错。加上 `&` 后，你匹配的是 `&Option<String>`，`time` 的类型变成 `&String`（引用），不会发生移动。

> [!HEADS-UP]
> 这个 `&t.completed_at` 是一个容易忽略的细节。如果你看到 `cannot move out of 't.completed_at' which is behind a shared reference`，原因就是你试图通过不可变引用移动一个非 `Copy` 的值。解决方法就是在 `match` 中借用：`match &t.completed_at`。这条规则对任何非 `Copy` 类型（`String`、`Vec`、`DateTime` 等）都适用。

## 拆分代码到多个模块

`main.rs` 现在有大约 250 行。对很多语言来说这不算长，但 Rust 的惯例是更早拆分——因为 Rust 的类型系统和编译检查意味着你会频繁在文件间跳转查看类型定义，文件短一些让跳转更快。

Rust 的模块系统由三个核心概念组成：`mod`（声明模块）、`pub`（标记公开项）、`use`（导入使用）。先解释概念，再动手拆分。

### mod：声明一个模块

`mod` 告诉编译器："这个名称对应一个模块，里面有一些代码。" 模块可以内联定义，也可以从文件加载：

```rust
// 内联模块——代码直接写在花括号里
mod math {
    fn add(a: i32, b: i32) -> i32 { a + b }
}

// 文件模块——只写分号，编译器去文件系统找
mod todo;   // 编译器查找 src/todo.rs 或 src/todo/mod.rs
```

对于项目拆分，文件模块是标准做法。当你在 `src/main.rs` 中写 `mod todo;`，编译器去找 `src/todo.rs`——文件名必须和模块名匹配。

### pub：标记公开项

Rust 中**所有项默认私有**。函数、结构体、枚举、常量——定义在模块里的项，只有同一个模块内的代码能访问。想让外部模块使用，必须加 `pub`：

```rust
// todo.rs
pub struct Todo { ... }     // 外部可以使用 Todo
pub fn print_todos(...) {}  // 外部可以调用 print_todos
fn internal_helper() {}     // 只有 todo.rs 内部能调用
```

结构体有一个额外的规则：即使结构体本身是 `pub`，它的**字段**默认仍然私有。你需要给每个需要外部访问的字段单独加 `pub`：

```rust
pub struct Todo {
    pub id: u32,              // 外部可以读写
    pub content: String,      // 外部可以读写
    pub completed: bool,      // 外部可以读写
    pub created_at: DateTime<Local>,
    pub priority: u8,
    pub completed_at: Option<String>,
}
```

> [!HEADS-UP]
> 这是初学者拆分模块时最常见的编译错误来源。你会看到 `field 'id' of struct 'Todo' is private`——结构体加了 `pub`，但字段忘了加。记住：`pub struct` 让**类型名**公开，`pub field` 让**字段**公开，两者是独立的开关。

枚举的规则比结构体简单：`pub enum` 的变体自动公开，不需要逐个标记。你在 `cli.rs` 中定义的 `pub enum Commands` 的所有变体（`Add`、`List` 等）自动可以在 `main.rs` 中使用。

### use：导入并使用

`use` 把其他模块的项引入当前作用域，让你不必每次都写完整路径：

```rust
// 不用 use——每次都要写完整路径
fn main() {
    let todo = todo::Todo::new(1, "hello".to_string(), 0);
    todo::print_todos("All", &[]);
    storage::save_todos(&[]).unwrap();
}

// 用 use——引入后直接用名称
use todo::{Todo, print_todos};
use storage::save_todos;

fn main() {
    let todo = Todo::new(1, "hello".to_string(), 0);
    print_todos("All", &[]);
    save_todos(&[]).unwrap();
}
```

`mod` 和 `use` 是两个不同的步骤，初学者容易混淆：`mod todo;` **声明**模块存在（告诉编译器去加载 `src/todo.rs`），`use todo::Todo;` **导入**模块中的具体项（在当前作用域创建快捷名称）。你需要先 `mod` 再 `use`。

### 动手拆分

把 `tasky` 拆成三个模块文件：`todo.rs`（数据模型）、`storage.rs`（文件读写）、`cli.rs`（CLI 定义）。命令处理函数（`cmd_add`、`cmd_list` 等）留在 `main.rs`——它们需要使用所有模块的类型，放在入口文件中最方便。

**第一步：创建 `src/todo.rs`**——Todo 结构体、构造函数、打印函数：

```rust
use chrono::{DateTime, Local};
use colored::Colorize;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Todo {
    pub id: u32,
    pub content: String,
    pub completed: bool,
    pub created_at: DateTime<Local>,
    #[serde(default)]
    pub priority: u8,
    #[serde(default)]
    pub completed_at: Option<String>,
}

impl Todo {
    pub fn new(id: u32, content: String, priority: u8) -> Self {
        Todo {
            id,
            content,
            completed: false,
            created_at: Local::now(),
            priority,
            completed_at: None,
        }
    }
}

pub fn print_todos(label: &str, items: &[&Todo]) {
    if items.is_empty() {
        println!("Nothing here.");
        return;
    }
    println!("  {} ({}):", label, items.len());
    for t in items {
        let priority_tag = match t.priority {
            2 => format!("  {}", "!! urgent".red().bold()),
            1 => format!("  {}", "! high".yellow()),
            _ => String::new(),
        };
        if t.completed {
            let completed_time = match &t.completed_at {
                Some(time) => format!("  ({})", time),
                None => String::new(),
            };
            println!(
                "    {} {}  {}{}  ({}){}",
                format!("[{}]", t.id).dimmed(),
                t.content.strikethrough(),
                "✓ done".green(),
                completed_time,
                t.created_at.format("%Y-%m-%d %H:%M"),
                priority_tag,
            );
        } else {
            println!(
                "    [{}] {}  ({}){}",
                t.id,
                t.content,
                t.created_at.format("%Y-%m-%d %H:%M"),
                priority_tag,
            );
        }
    }
}
```

注意每个文件顶部的 `use` 语句——每个模块文件需要**自己**的导入。`todo.rs` 需要 `chrono`（因为 `DateTime<Local>`）、`colored`（因为 `.green()` 等方法）、`serde`（因为 `Serialize`/`Deserialize`）。`main.rs` 中的 `use` 语句不会被 `todo.rs` 继承——每个文件是独立的作用域。

`pub fn new` 中的 `pub` 让 `main.rs` 可以调用 `Todo::new()`。没有 `pub`，`impl` 块中的方法默认是私有的——只能在 `todo.rs` 内部调用。

**第二步：创建 `src/storage.rs`**——文件读写函数：

```rust
use anyhow::{Context, Result};
use std::fs;
use std::path::PathBuf;

use crate::todo::Todo;

fn data_file() -> PathBuf {
    let dir = dirs::config_dir()
        .expect("Cannot determine config directory")
        .join("tasky");
    fs::create_dir_all(&dir).expect("Cannot create config directory");
    dir.join("todos.json")
}

pub fn load_todos() -> Result<Vec<Todo>> {
    let path = data_file();
    if !path.exists() {
        return Ok(Vec::new());
    }
    let content = fs::read_to_string(&path)
        .context("Failed to read todos file")?;
    let todos: Vec<Todo> = serde_json::from_str(&content)
        .unwrap_or_default();
    Ok(todos)
}

pub fn save_todos(todos: &[Todo]) -> Result<()> {
    let path = data_file();
    let json = serde_json::to_string_pretty(todos)
        .context("Failed to serialize todos")?;
    fs::write(&path, json)
        .context("Failed to write todos file")?;
    Ok(())
}
```

这里有一个新概念：`use crate::todo::Todo;`。`crate` 指向整个项目的根——也就是 `main.rs`。`crate::todo::Todo` 意思是"从项目根的 `todo` 模块中取 `Todo`"。`storage.rs` 需要 `Todo` 类型来声明函数签名（`load_todos` 返回 `Vec<Todo>`，`save_todos` 接受 `&[Todo]`），所以必须导入它。

注意 `data_file` **没有** `pub`——它是 `storage` 模块的私有实现细节。`load_todos` 和 `save_todos` 在同一个模块内可以调用它，但 `main.rs` 看不到也不需要看到 `data_file`。这就是模块化的好处：隐藏实现细节，暴露最小接口。如果你将来想把存储从 JSON 文件换成 SQLite 数据库，只需要修改 `storage.rs` 内部——`main.rs` 中的调用代码完全不需要变。

> [!ASIDE]
> **`super::` 关键字。** 除了 `crate::`（从项目根开始的路径），你还可以用 `super::` 从**父模块**开始寻址。如果 `todo.rs` 中有一个子模块 `todo::validation`，在 `validation` 中访问 `todo.rs` 的项可以用 `super::Todo`（`super` 指向父模块 `todo`）。对于我们的扁平结构（所有模块文件都是 `main.rs` 的直接子模块），`crate::` 和 `super::` 效果相同——但 `crate::` 更明确，是推荐的写法。

**第三步：创建 `src/cli.rs`**——CLI 定义：

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "tasky", version, about = "A tiny todo manager")]
pub struct Cli {
    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Subcommand)]
pub enum Commands {
    /// Add a new todo
    Add {
        /// The task description
        #[arg(required = true)]
        content: String,
        /// Priority: 0 = normal, 1 = high, 2 = urgent
        #[arg(short, long, default_value_t = 0)]
        priority: u8,
    },
    /// Edit a todo
    Edit {
        /// The ID of the todo to edit
        #[arg(required = true)]
        id: u32,
        /// The new content
        #[arg(required = true)]
        content: String,
    },
    /// List todos (pending by default)
    List {
        /// Show all todos including completed
        #[arg(short, long)]
        all: bool,
        /// Filter by minimum priority
        #[arg(short, long)]
        priority: Option<u8>,
    },
    /// Mark a todo as done
    Done {
        /// The ID of the todo to complete
        id: u32,
        /// Undo the completion
        #[arg(long)]
        undo: bool,
    },
    /// Remove a todo
    Remove {
        /// The ID of the todo to remove
        id: u32,
    },
    /// Search todos by keyword
    Search {
        /// The keyword to search for
        keyword: String,
    },
}
```

`pub struct Cli`、`pub struct Commands`（等等，`Commands` 是 `pub enum`）——`pub enum Commands` 的变体自动公开，不需要每个变体单独标 `pub`。这和结构体字段的规则不同，记住这个区别。

**第四步：重写 `src/main.rs`**——只保留命令路由和命令处理函数：

```rust
use anyhow::Result;
use chrono::Local;
use clap::Parser;
use colored::Colorize;

mod cli;
mod storage;
mod todo;

use cli::{Cli, Commands};
use storage::{load_todos, save_todos};
use todo::{print_todos, Todo};

fn cmd_add(todos: &mut Vec<Todo>, content: String, priority: u8) {
    let new_id = todos.iter().map(|t| t.id).max().unwrap_or(0) + 1;
    let todo = Todo::new(new_id, content, priority);
    let priority_info = if priority > 0 {
        format!(
            "  [{}]",
            match priority {
                2 => "urgent".red().bold().to_string(),
                1 => "high".yellow().to_string(),
                _ => "normal".to_string(),
            }
        )
    } else {
        String::new()
    };
    println!(
        "{} Added #{}: {}{}",
        "✓".green(),
        todo.id,
        todo.content,
        priority_info
    );
    todos.push(todo);
}

fn cmd_list(todos: &[Todo], all: bool, priority: Option<u8>) {
    let items: Vec<&Todo> = todos
        .iter()
        .filter(|t| all || !t.completed)
        .filter(|t| match priority {
            Some(p) => t.priority >= p,
            None => true,
        })
        .collect();
    let label = if all { "All" } else { "Pending" };
    print_todos(label, &items);
}

fn cmd_done(todos: &mut Vec<Todo>, id: u32, undo: bool) {
    match todos.iter_mut().find(|t| t.id == id) {
        Some(todo) => {
            let sign = if undo { "Undo" } else { "Done" };
            todo.completed = !undo;
            todo.completed_at = if undo {
                None
            } else {
                Some(Local::now().format("%Y-%m-%d %H:%M").to_string())
            };
            println!("{}{} #{}: {}", "✓".green(), sign, id, todo.content);
        }
        None => {
            eprintln!("Todo #{} not found.", id);
            std::process::exit(1);
        }
    }
}

fn cmd_edit(todos: &mut Vec<Todo>, id: u32, content: &str) {
    match todos.iter_mut().find(|t| t.id == id) {
        Some(todo) => {
            todo.content = content.to_string();
            println!("{} #{}: {}", "✓ Edit".yellow(), id, content);
        }
        None => {
            eprintln!("Todo #{} not found.", id);
            std::process::exit(1);
        }
    }
}

fn cmd_remove(todos: &mut Vec<Todo>, id: u32) {
    let len_before = todos.len();
    todos.retain(|t| t.id != id);
    if todos.len() < len_before {
        println!("Removed #{}", id);
    } else {
        eprintln!("Todo #{} not found.", id);
        std::process::exit(1);
    }
}

fn cmd_search(todos: &[Todo], keyword: &str) {
    let matches: Vec<&Todo> = todos
        .iter()
        .filter(|t| t.content.contains(keyword))
        .collect();
    let label = format!("Search: \"{}\"", keyword);
    print_todos(&label, &matches);
}

fn main() -> Result<()> {
    let cli = Cli::parse();
    let mut todos = load_todos()?;

    match cli.command {
        Commands::Add { content, priority } => {
            cmd_add(&mut todos, content, priority);
            save_todos(&todos)?;
        }
        Commands::List { all, priority } => {
            cmd_list(&todos, all, priority);
        }
        Commands::Done { id, undo } => {
            cmd_done(&mut todos, id, undo);
            save_todos(&todos)?;
        }
        Commands::Remove { id } => {
            cmd_remove(&mut todos, id);
            save_todos(&todos)?;
        }
        Commands::Search { keyword } => {
            cmd_search(&todos, &keyword);
        }
        Commands::Edit { id, content } => {
            cmd_edit(&mut todos, id, &content);
            save_todos(&todos)?;
        }
    }

    Ok(())
}
```

`main.rs` 的结构现在非常清晰：顶部 `mod` 声明三个子模块，然后 `use` 导入需要的项，中间是六个命令处理函数，最后是 `main` 函数做路由。整个文件不到 120 行。

注意 `mod` 语句必须出现在 `use` 语句之前——你得先声明模块存在，才能从中导入项。编译器按文件从上到下处理，如果 `use` 出现在 `mod` 之前，编译器会报 `unresolved import` 错误。

> [!DESIGN-NOTE]
> **`use cli::*;` 可以吗？** 可以。`*` 是 glob 操作符，导入模块中所有公开项。对于 `Commands` 枚举（你经常需要引用所有变体），`use cli::*;` 是合理的简写。但在大型项目中，glob 导入会让"这个名称从哪来的"变得难以追踪。Rust 社区的惯例是：对 `enum` 变体用 glob（`use cli::Commands::*;` 导入所有变体），对结构体和函数用显式导入（`use cli::Cli;`）。

### 拆分后的项目结构

```
tasky/
├── Cargo.toml
└── src/
    ├── main.rs       ← mod 声明 + use 导入 + 命令处理 + main()
    ├── todo.rs       ← Todo 结构体 + impl + print_todos
    ├── storage.rs    ← data_file + load_todos + save_todos
    └── cli.rs        ← Cli 结构体 + Commands 枚举
```

当你在 `main.rs` 中写 `mod todo;`，编译器在 `src/` 下找 `todo.rs`。如果你需要更深的嵌套——比如在 `todo` 模块下创建一个 `validation` 子模块——可以这样组织：

```
src/
├── main.rs
├── todo.rs           ← mod validation;
└── todo/
    └── validation.rs ← validation 子模块的代码
```

`todo.rs` 中写 `mod validation;`，编译器查找 `src/todo/validation.rs`。这和你从 `main.rs` 加载 `src/todo.rs` 的规则完全相同——只是基目录从 `src/` 变成了 `src/todo/`。

> [!ASIDE]
> **模块树的心智模型。** Rust 的所有模块构成一棵树，树根是 `crate`（隐式的，对应 `src/main.rs`）。`mod todo;`、`mod storage;`、`mod cli;` 在树根下创建了三个子节点。每个子节点内部又可以有自己的子模块。理解这棵树有助于你理解路径：`crate::todo::Todo` 是沿着树从根走到 `todo` 再到 `Todo`；`crate::storage::load_todos` 是从根走到 `storage` 再到 `load_todos`。所有路径解析问题，本质上都是在这棵树上找节点。

## 集成测试：验证 CLI 行为

到目前为止，你测试 `tasky` 的方式是手动运行命令、目视检查输出。这在开发阶段够用，但随着功能增多，你需要一种**可重复、可自动化**的方式来验证行为——这就是测试的用途。

Rust 有两种测试。**单元测试**（unit tests）和被测试的代码放在同一个文件中，用 `#[cfg(test)]` 标注的模块里，测试函数内部的细节。**集成测试**（integration tests）放在项目根目录的 `tests/` 文件夹中，每个 `.rs` 文件是一个独立的 crate，测试程序作为一个整体的行为。

对于 `tasky` 这样的 CLI 工具，集成测试更有价值——你想验证"运行 `tasky add 'xxx'` 后输出包含 `Added`"，而不是某个内部函数的返回值。

### 用 assert_cmd 测试二进制程序

`assert_cmd` crate 让你可以在测试中启动 `tasky` 的编译产物，传入命令行参数，然后断言输出和退出码。你在前置条件中已经添加了它和 `predicates`：

```toml
[dev-dependencies]
assert_cmd = "2"
predicates = "3"
```

`[dev-dependencies]` 中的依赖**只在** `cargo test` 时编译——它们不会增加你的发布二进制的大小。

创建 `tests/cli_test.rs`：

```rust
use assert_cmd::Command;
use predicates::prelude::*;
use tempfile::TempDir;

fn setup_test_home() -> TempDir {
    let tmp = tempfile::tempdir().unwrap();
    unsafe {
        std::env::set_var("HOME", tmp.path());
    }
    tmp
}
```

等等——`tempfile` 是一个新的 crate。你需要把它也加到 `[dev-dependencies]`：

```toml
[dev-dependencies]
assert_cmd = "2"
predicates = "3"
tempfile = "3"
```

`tempfile::tempdir()` 创建一个临时目录，测试结束后自动删除。`set_var("HOME", ...)` 把 `HOME` 环境变量指向临时目录，这样 `dirs::config_dir()` 会在临时目录里创建 `tasky/todos.json`——你的测试不会影响真实的待办数据。

> [!HEADS-UP]
> **为什么 `set_var` 需要 `unsafe`？** 从 Rust edition 2024 开始，`std::env::set_var` 被标记为 `unsafe fn`——因为修改环境变量**不是线程安全的**（多个线程同时读写环境变量可能导致未定义行为）。在 edition 2021 中这个函数不需要 `unsafe`，但你的 `Cargo.toml` 中 `edition = "2024"`，所以必须用 `unsafe {}` 块包裹调用。`unsafe` 块的含义是"我（程序员）保证这段代码在当前上下文中是安全的"——在测试里串行运行时，这个保证是成立的。

> [!HEADS-UP]
> `set_var` 修改的是**当前进程**的环境变量。`cargo test` 默认并行运行测试，多个测试同时修改 `HOME` 会互相干扰。解决方法有两种：一是用 `cargo test -- --test-threads=1` 串行运行，二是在每个测试中使用不同的环境变量（比如设置 `TASKY_DATA_DIR` 并在代码中读取）。对于入门级测试，`--test-threads=1` 是最简单的方案。

现在写测试函数。每个测试用 `#[test]` 注解标记：

```rust
#[test]
fn test_add_and_list() {
    let _tmp = setup_test_home();

    // 添加一条待办
    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["add", "测试任务"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Added #1"));

    // 列出待办
    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["list"])
        .assert()
        .success()
        .stdout(predicate::str::contains("测试任务"));
}

#[test]
fn test_done_marks_completed() {
    let _tmp = setup_test_home();

    // 先添加
    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["add", "完成任务"])
        .assert()
        .success();

    // 标记完成
    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["done", "1"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Done #1"));

    // 默认 list 不应显示已完成的
    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["list"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Nothing here"));
}

#[test]
fn test_remove_nonexistent() {
    let _tmp = setup_test_home();

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["remove", "99"])
        .assert()
        .failure();
}
```

拆解 `Command::cargo_bin("tasky").unwrap().args(&["add", "测试任务"]).assert()` 这个链条：

1. `Command::cargo_bin("tasky")` — 定位 `cargo build` 编译出的 `tasky` 二进制文件，返回 `Result<Command>`
2. `.unwrap()` — 提取 `Command`（如果找不到二进制文件会 panic——这只在编译失败时发生）
3. `.args(&["add", "测试任务"])` — 设置命令行参数，等价于运行 `tasky add "测试任务"`
4. `.assert()` — 执行命令并返回一个断言对象
5. `.success()` — 断言退出码为 0（程序正常结束）
6. `.stdout(predicate::str::contains("Added #1"))` — 断言标准输出包含指定文本

`predicates` crate 提供了一套灵活的匹配器：`predicate::str::contains("text")` 检查包含关系，`predicate::str::is_empty()` 检查空字符串，`predicate::eq(42)` 检查精确相等。你可以在 `.stdout()` 和 `.stderr()` 中使用它们。

运行测试：

```bash
cargo test -- --test-threads=1
```

预期输出：

```
running 3 tests
test test_add_and_list ... ok
test test_done_marks_completed ... ok
test test_remove_nonexistent ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

`--test-threads=1` 确保测试串行运行，避免 `HOME` 环境变量冲突。

> [!DESIGN-NOTE]
> **二进制 crate 的测试限制。** Rust 的集成测试通过 `use` 导入库 crate 的公开项来测试。但 `tasky` 是二进制 crate（只有 `main.rs`，没有 `lib.rs`），不能被其他 crate `use`。`assert_cmd` 绕过了这个限制——它直接运行编译出的二进制文件，像真实用户一样传参数、检查输出。这是测试 CLI 工具最自然的方式。如果你想在测试中直接调用 `Todo::new()` 或 `load_todos()` 这样的内部函数，需要创建 `src/lib.rs` 把逻辑导出为库——但那是后续话题。

## cargo install：安装你的工具

你的 `tasky` 已经可以用了——但每次都要 `cargo run --` 才能执行，而且只能在项目目录下运行。`cargo install` 让你把它安装为全局命令。

在项目根目录运行：

```bash
cargo install --path .
```

Cargo 编译 release 版本（优化后的二进制，比 `cargo build` 的 debug 版本更快、更小），然后把 `tasky` 安装到 `~/.cargo/bin/` 目录。如果 `~/.cargo/bin/` 在你的 `$PATH` 中（通过 rustup 安装 Rust 时通常已自动配置），你可以在**任何目录**直接使用：

```bash
cd ~/some/other/project
tasky add "不用 cargo run 了"
tasky list
```

想确认安装位置？运行 `which tasky`（macOS/Linux）或 `where tasky`（Windows），应该输出类似 `/Users/你的名字/.cargo/bin/tasky` 的路径。

### 卸载你的工具

`tasky` 是学习项目，学完之后你可能想把它从系统中移除。卸载分两步：

```bash
# 第一步：删除二进制文件
cargo uninstall tasky
```

`cargo uninstall` 会删除 `~/.cargo/bin/tasky`，之后 `tasky` 命令不再可用。

```bash
# 第二步（可选）：删除数据文件
rm -rf ~/Library/Application\ Support/tasky
```

这一步删除 `tasky` 存储的 `todos.json` 数据文件——也就是你用 `tasky add` 添加的那些待办事项。如果你将来想重新安装并保留旧数据，可以跳过这步。

> [!ASIDE]
> **`cargo install` vs 包管理器。** 你大概注意到了，`cargo install` 没有提供 `cargo list`（列出已安装的 crate）或批量卸载功能。如果你安装了多个工具想统一管理，可以查看 `~/.cargo/bin/` 目录下的所有文件，或者使用社区工具 [`cargo-binstall`](https://github.com/cargo-bins/cargo-binstall) 来管理预编译的二进制安装。对于 `tasky` 这样的单个工具，`cargo uninstall` 足够了。

### 发布到 crates.io

如果你希望**任何人**都能用 `cargo install tasky` 安装你的工具，需要把它发布到 [crates.io](https://crates.io)——Rust 的官方包注册中心。

先完善 `Cargo.toml` 的元数据：

```toml
[package]
name = "tasky"
version = "0.1.0"
edition = "2024"
description = "A tiny command-line todo manager"
license = "MIT"
authors = ["Your Name <your@email.com>"]
```

`description` 和 `license` 是 crates.io 发布所必需的。`MIT` 是最常见的开源许可证之一——允许任何人自由使用、修改和分发你的代码，只要保留版权声明。

然后注册账号、登录、发布：

```bash
# 在 https://crates.io/me 创建 API token
cargo login <your-api-token>

# 发布（会上传代码和元数据）
cargo publish
```

发布后，任何人运行 `cargo install tasky` 就能安装你的工具。

> [!HEADS-UP]
> crates.io 上的 crate 名称是全局唯一的——如果有人已经发布了叫 `tasky` 的 crate，你需要换一个名字。发布后无法删除版本（只能发布新版本标记旧版本为 yanked），所以在发布前确认名称和版本号是你满意的。

## 检查点

> [!PREDICT]
> 在运行之前想一想：拆分模块后，`load_todos` 从 `main.rs` 移到了 `storage.rs`。但 `main.rs` 中的调用代码 `load_todos()?` 看起来完全没变。为什么不需要改成 `storage::load_todos()?`？

**运行以下命令验证你目前的工作：**

首先，确保模块文件都已创建：

```bash
ls src/
```

预期输出（四个文件）：

```
cli.rs  main.rs  storage.rs  todo.rs
```

```bash
cargo build
```

预期输出：编译成功，无错误。

```bash
cargo run -- add "准备周报"
```

预期输出：

```
✓ Added #1: 准备周报
```

```bash
cargo run -- add -p 2 "修复线上 bug"
```

预期输出：

```
✓ Added #2: 修复线上 bug  [urgent]
```

```bash
cargo run -- done 1
```

预期输出：

```
✓ Done #1: 准备周报
```

```bash
cargo run -- list --all
```

预期输出（注意已完成条目的完成时间）：

```
  All (2):
    [1] 准备周报  ✓ done  (2026-06-14 17:30)  (2026-06-14 09:00)
    [2] 修复线上 bug  (2026-06-14 09:15)  !! urgent
```

```bash
cargo run -- done --undo 1
```

预期输出：

```
✓ Undo #1: 准备周报
```

```bash
cargo run -- list
```

预期输出（撤销完成后，`completed_at` 被清除，条目重新出现在 pending 列表中）：

```
  Pending (2):
    [1] 准备周报  (2026-06-14 09:00)
    [2] 修复线上 bug  (2026-06-14 09:15)  !! urgent
```

```bash
cargo test -- --test-threads=1
```

预期输出：所有测试通过。

**可能的错误：**

- 如果看到 `cannot find type 'Todo' in this scope`（在 `main.rs` 中），你可能忘了在 `main.rs` 顶部添加 `use todo::Todo;`——`mod todo;` 只声明模块存在，不自动导入任何项。
- 如果看到 `field 'id' of struct 'Todo' is private`，你可能给 `Todo` 加了 `pub` 但忘了给字段加 `pub`——每个需要外部访问的字段都需要单独的 `pub`。
- 如果看到 `unresolved import 'crate::todo::Todo'`（在 `storage.rs` 中），你可能在 `main.rs` 中忘了 `mod todo;`——`storage.rs` 通过 `crate::todo` 路径访问 `Todo`，前提是 `main.rs` 声明了 `mod todo;`。
- 如果看到 `use of undeclared crate or module 'colored'`（在 `todo.rs` 中），你可能忘了在 `todo.rs` 顶部添加 `use colored::Colorize;`——每个模块文件需要自己的 `use` 语句。
- 如果看到 `cannot find function 'Todo::new' in this scope`，你可能忘了给 `impl Todo` 中的 `fn new` 加 `pub`——方法默认是私有的。

## 接下来

你的 `tasky` 现在结构清晰：数据模型、存储逻辑、CLI 定义、命令处理各居其所。你可以用 `cargo test` 自动验证行为，用 `cargo install --path .` 全局安装。

回顾三个部分的学习路径：part 1 教你用结构体、枚举、`match` 和文件 I/O 构建一个能用的 CLI；part 2 引入 trait（`Colorize`）、`Option`（在 `priority` 参数中）和 `Result`（用 `anyhow` 处理错误）让你处理更复杂的场景；part 3 深入 `Option<T>` 的模式匹配、模块系统的可见性控制、和测试框架。这些概念不是孤立的——你在每个新场景中反复遇到 `match`、借用、迭代器链，直到它们变成肌肉记忆。

后续可以探索的方向包括：异步编程（用 `tokio` 让 `tasky` 支持网络同步）、TUI 界面（用 `ratatui` 构建终端交互界面）、或者把 `tasky` 的核心逻辑重构为库 crate（创建 `lib.rs`）让其他程序也能使用你的待办管理逻辑。

在继续之前，用你自己的话回答：`mod` 和 `use` 的区别是什么？如果你只写了 `mod todo;` 但没有 `use todo::Todo;`，在 `main.rs` 中直接使用 `Todo` 会发生什么？如果你只写了 `use todo::Todo;` 但没有 `mod todo;`，又会发生什么？

## 练习

1. **给 `Todo` 添加 `tags` 字段。** 在结构体中添加 `#[serde(default)] tags: Vec<String>`。`Vec<String>` 表示一个字符串列表——一个待办可以有多个标签。给 `add` 命令添加 `#[arg(short, long)] tag: Vec<String>` 选项，用户可以多次传 `--tag`（如 `tasky add "学习 Rust" --tag learning --tag rust`）。在 `print_todos` 中把标签显示在优先级之后，用 `colored` 的 `.cyan()` 着色。这练习 `Vec` 作为字段类型和 clap 参数的用法——`clap` 会自动把多次出现的选项收集到 `Vec` 中。

2. **添加 `stats` 子命令。** 显示待办统计信息：总数、已完成数、各优先级的数量。创建 `src/commands.rs`，把 `stats` 函数放在其中，用 `pub fn stats(todos: &[Todo])` 导出。这练习创建新模块文件——别忘了在 `main.rs` 中加 `mod commands;`。

3. **给 `storage.rs` 添加单元测试。** 在 `storage.rs` 底部添加 `#[cfg(test)]` 模块，写一个测试验证 `save_todos` 和 `load_todos` 的往返一致性：保存一些 `Todo`，加载回来，断言数据相同。`#[cfg(test)]` 模块只在 `cargo test` 时编译，不影响正常构建。

4. **实现 `clear` 子命令。** 删除所有**已完成**的待办（保留未完成的）。提示：`Vec::retain(|t| !t.completed)` 就地保留未完成的条目。注意 `clear` 需要 `&mut Vec<Todo>` 并且需要调用 `save_todos`。

5. **给 `tasky` 添加 `--data-dir` 全局选项。** 让用户可以通过 `tasky --data-dir /tmp/test add "xxx"` 指定数据存储目录。在 `Cli` 结构体（不是 `Commands` 枚举）中添加一个字段，然后把它传递给 `load_todos` 和 `save_todos`（替换 `dirs::config_dir()` 的默认路径）。完成后你的集成测试可以用 `--data-dir` 指向临时目录，不再需要修改 `HOME` 环境变量——测试更可靠。

## 来源

1. [The Rust Programming Language — Chapter 7: Managing Growing Projects](https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html) — 模块系统的官方教程，`mod`、`pub`、`use` 的权威参考。
2. [The Rust Programming Language — Chapter 6: Enums and Pattern Matching](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html) — `Option<T>` 枚举的定义和 `match` 模式匹配。
3. [The Rust Programming Language — if let](https://doc.rust-lang.org/book/ch06-03-if-let.html) — `if let` 语法糖的详细解释和与 `match` 的对比。
4. [std::option::Option documentation](https://doc.rust-lang.org/std/option/enum.Option.html) — `Option<T>` 所有方法的完整文档，包括 `unwrap_or`、`map`、`and_then`、`is_some` 等。
5. [The Rust Programming Language — Chapter 11: Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html) — 单元测试和集成测试的目录约定。
6. [assert_cmd documentation](https://docs.rs/assert_cmd/latest/assert_cmd/) — CLI 集成测试框架，`Command::cargo_bin` 和断言 API。
7. [The Rust Programming Language — Installing Binaries](https://doc.rust-lang.org/book/ch14-04-installing-binaries.html) — `cargo install` 的使用和二进制 crate 的分发。
8. [predicates documentation](https://docs.rs/predicates/latest/predicates/) — 断言匹配器库，与 `assert_cmd` 配合使用。
9. [The Rust Programming Language — Bringing Paths into Scope with use](https://doc.rust-lang.org/book/ch07-04-bringing-paths-into-scope-with-the-use-keyword.html) — `use` 关键字的最佳实践、`pub use` 重导出和嵌套路径。
