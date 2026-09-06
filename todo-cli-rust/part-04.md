# 用 Rust 构建命令行待办事项应用 — 第四部分

> [!RECALL]
> 在 part 3 中，你把 `main.rs` 拆分成了 `todo.rs`、`storage.rs`、`cli.rs` 三个模块文件。回忆一下：`storage.rs` 中想使用 `Todo` 类型，写的是 `use crate::todo::Todo;` 而不是 `use todo::Todo;`。`crate::` 这个前缀的含义是什么？如果 `main.rs` 中忘了写 `mod todo;`，这条 `use` 语句会报什么错？

经过三个部分的学习，你的 `tasky` 已经是一个功能完整的 CLI 工具：七个子命令、优先级和标签分类、完成时间记录、模块化的代码结构、基本的集成测试。part 3 末尾留了五道练习，你已经完成了前两题（`tags` 字段和 `stats` 命令），还有三道等着我们。

本部分先做一轮核心概念回顾——距离 part 1 已经过了一段时间，所有权、借用、`match`、迭代器这些概念你可能需要重新激活。然后逐步完成剩余的三道练习（`clear` 命令、`--data-dir` 全局参数、单元测试），最后引入一个新主题：为你的自定义类型实现标准库 trait。

本部分结束时，你的 `tasky` 将拥有完整的练习覆盖、可配置的存储路径、以及一个实现了 `Display` trait 的 `Todo` 类型——你可以直接用 `{}` 格式化打印待办事项。

## 你将构建什么

```
$ tasky --data-dir /tmp/mytasks add "用临时目录测试"
✔ Added #1: 用临时目录测试

$ tasky stats
待办总数为：1，已经完成的任务数为：0, 未完成数量：1
各个优先级待办数量：普通待办:1 高等级待办:0 紧急待办:0

$ tasky done 1
✔ Done #1: 用临时目录测试

$ tasky clear
✓ Cleared 1 completed todo(s)
```

新增的能力：`clear` 命令批量清理已完成的待办，`--data-dir` 全局参数让你指定数据存储目录（测试和日常使用分离），单元测试验证 `storage.rs` 的读写逻辑。

## 前置条件

- part 1-3 完成的代码，已完成 part 3 练习 1（`tags`）和练习 2（`stats`）
- 如果你的代码还没有完成这两题，先参照 part 3 的练习说明补上

## 核心概念回顾

距离 part 1 已经过了一段时间。在你继续写新代码之前，让我们快速回顾三个部分中反复出现的核心概念。这不是从零讲解——而是帮你重新激活已有的知识，让你在后面的练习中更快进入状态。

### 所有权与借用

Rust 最独特的特性。每个值有且只有一个所有者，当所有者离开作用域时值被释放。你在 `cmd_add` 中已经反复体验了这个规则：

```rust
fn cmd_add(todos: &mut Vec<Todo>, content: String, priority: u8, tags: Vec<String>) {
    // content 和 tags 的所有权在函数调用时从 main() 转移到 cmd_add
    let todo = Todo::new(new_id, content, priority, tags);
    // content 和 tags 的所有权又转移给了 Todo::new
    // 此后再使用 content 或 tags 会报错 "value used after move"
    todos.push(todo);  // todo 的所有权转移给 Vec
}
```

**借用**让你在不获取所有权的情况下访问值。`&T` 是不可变借用（只读），`&mut T` 是可变借用（可读写）。你在 `cmd_list` 的参数 `todos: &[Todo]` 中使用了不可变借用——列表函数不需要修改待办数据，也不需要获取所有权。

```rust
fn cmd_list(todos: &[Todo], ...) { ... }    // 不可变借用：只读
fn cmd_add(todos: &mut Vec<Todo>, ...) { ... } // 可变借用：需要修改
```

**核心规则：** 同一时刻只能有一个 `&mut` 借用，或者多个 `&` 借用——不能同时有 `&mut` 和 `&`。你在 part 2 中遇到过的 `iter_mut().find()` 返回的 `Some(&mut Todo)` 就是这个规则的体现——在 `find` 持有可变引用的期间，你不能再对 `todos` 做其他操作。

### match 与模式匹配

`match` 是你最常用的控制流工具之一。你在三个部分中用它处理了四种不同的枚举：

```rust
// 1. Commands 枚举（part 1）—— 路由子命令
match cli.command {
    Commands::Add { content, priority, tags } => { ... }
    Commands::List { all, priority, tag } => { ... }
    ...
}

// 2. Option<T>（part 2-3）—— 处理可能不存在的值
match &t.completed_at {
    Some(time) => format!(" ({})", time.format("%Y-%m-%d %H:%M")),
    None => String::new(),
}

// 3. Result<T, E>（part 2）—— 通过 ? 操作符隐式匹配
let todos = load_todos()?;  // 隐含 match：Ok(t) => t, Err(e) => return Err(e)

// 4. u8 值匹配（part 1-3）—— 优先级分类
match t.priority {
    2 => "!!urgent".red().bold(),
    1 => "! high".yellow(),
    _ => String::new(),  // 通配分支
}
```

`match` 的核心规则：**穷举**——编译器强制你覆盖所有可能的模式，遗漏一个变体就会报编译错误。`_` 通配符用于"其余所有情况"。

`if let` 是 `match` 的简写——当你只关心一个模式时使用：

```rust
// 这两种写法等价：
match &tag { Some(p) => ..., _ => () }
if let Some(p) = &tag { ... }
```

### 迭代器链

你在 `cmd_list` 和 `cmd_search` 中构建的过滤链，是 Rust 函数式编程风格的核心模式：

```rust
let items: Vec<&Todo> = todos
    .iter()                         // 1. 创建迭代器（惰性）
    .filter(|t| all || !t.completed) // 2. 过滤：保留未完成的
    .filter(|t| match priority {     // 3. 再过滤：按优先级
        Some(p) => t.priority >= p,
        None => true,
    })
    .collect();                      // 4. 消费：收集为 Vec
```

关键点：`.iter()` 创建的迭代器是**惰性**的——它只是描述"要怎么遍历"，不做任何计算。直到 `.collect()`（或 `.count()`、`for` 循环）被调用时，闭包才会实际执行。你在 part 3 练习中踩过的坑（`.map()` 的闭包没有执行）就是这个惰性机制导致的。

`.map()` vs `.for_each()`：前者用于**变换**值（`|t| t.id` 把 `&Todo` 变成 `u32`），后者用于**执行副作用**（打印、计数、修改外部变量）。如果你不需要返回值，用 `.for_each()` 或 `for` 循环。

### Option<T> 和 Result<T, E>

这两个类型是 Rust 处理"不确定性"的核心工具。它们的结构非常相似：

| | Option\<T\> | Result\<T, E\> |
|---|---|---|
| **成功** | `Some(T)` | `Ok(T)` |
| **失败/缺失** | `None` | `Err(E)` |
| **用途** | 值可能存在也可能不存在 | 操作可能成功也可能失败 |
| **常用处理** | `match`、`if let`、`unwrap_or()` | `match`、`?`、`.context()` |

你在 `cmd_list` 的 `priority: Option<u8>` 参数中用了 `Option`，在 `load_todos() -> Result<Vec<Todo>>` 中用了 `Result`。

### 模块系统速查

| 关键字 | 含义 | 示例 |
|---|---|---|
| `mod` | 声明一个子模块 | `mod todo;`（加载 `src/todo.rs`） |
| `pub` | 标记项为公开 | `pub struct Todo`、`pub fn load_todos()` |
| `use` | 导入项到当前作用域 | `use todo::Todo;` |
| `crate::` | 从项目根开始的路径 | `use crate::todo::Todo;`（在子模块中） |
| `super::` | 从父模块开始的路径 | `use super::data_file;`（在同级子模块中） |

每个模块文件需要**自己的** `use` 语句——`main.rs` 中导入的 crate 不会自动在 `todo.rs` 中可用。

### trait 速查

你在 part 2 中第一次遇到 trait——`colored` crate 的 `Colorize` trait 为 `String` 和 `&str` 添加了 `.green()`、`.red()` 等方法。trait 是 Rust 中定义共享行为的方式，类似其他语言的接口（interface）。

```rust
// 你已经"使用"过的 trait：
use colored::Colorize;    // 给 String/&str 添加颜色方法
use serde::Serialize;     // 给结构体添加序列化为 JSON 的能力
use clap::Parser;         // 给结构体添加命令行解析的能力
use std::fmt::Display;    // 给类型添加 {} 格式化打印的能力（本部分新内容）
```

`#[derive(Serialize)]` 让编译器**自动实现** trait——你不需要手写实现代码。但不是所有 trait 都能 derive：`Display` 需要你自己写 `impl` 块，因为编译器不知道你想怎么"展示"一个待办事项。本部分最后一个章节会详细讲这个。

## clear 子命令：批量清理

part 3 的练习 4 要求实现 `clear` 命令——删除所有已完成的待办。

### 实现 clear

在 `cli.rs` 的 `Commands` 枚举中添加变体：

```rust
/// Remove all completed todos
Clear,
```

在 `main.rs` 中添加 `cmd_clear` 函数：

```rust
fn cmd_clear(todos: &mut Vec<Todo>) {
    let before = todos.len();
    todos.retain(|t| !t.completed);
    let removed = before - todos.len();
    if removed > 0 {
        println!("{} Cleared {} completed todo(s)", "✓".green(), removed);
    } else {
        println!("No completed todos to clear.");
    }
}
```

在 `main` 的 `match` 中添加路由分支：

```rust
Commands::Clear => {
    cmd_clear(&mut todos);
    save_todos(&todos)?;
}
```

`Vec::retain(|t| !t.completed)` 是一个就地操作——它修改原始 `Vec`，只保留使闭包返回 `true` 的元素。这里保留**未完成**的待办（`!t.completed`），等效于删除所有已完成的。

> [!DESIGN-NOTE]
> **为什么用 `retain` 而不是 `filter` + 重新赋值？** `.filter()` 返回一个迭代器，你需要 `.collect()` 生成新的 `Vec` 再赋值给 `todos`。`.retain()` 直接在原 `Vec` 上修改，不分配新内存——对大量数据更高效。两者的语义也不同：`retain` 表达"就地删除不符合条件的"，`filter` 表达"选择符合条件的组成新集合"。当你不需要旧数据时，`retain` 更直接。

## 全局参数：--data-dir

part 3 的练习 5 要求添加 `--data-dir` 全局选项。这个参数出现在**子命令之前**：

```bash
tasky --data-dir /tmp/test add "临时任务"
#       ^^^^^^^^^^^^^^^^^^^^^ 全局参数
#                              ^^^ 子命令
```

### Cli 结构体上的字段

在 part 3 中，你的 `Cli` 结构体只有一个字段：

```rust
#[derive(Parser)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Commands,
}
```

clap 允许在 `Cli` 上添加普通字段——这些字段成为**全局参数**，出现在任何子命令之前。添加 `data_dir`：

```rust
use std::path::PathBuf;

#[derive(Parser)]
#[command(name = "tasky", version, about = "A tiny todo manager")]
pub struct Cli {
    /// Custom data directory (overrides default)
    #[arg(long, global = true)]
    pub data_dir: Option<PathBuf>,
    #[command(subcommand)]
    pub command: Commands,
}
```

两个新要素：

**`global = true`** — 告诉 clap 这个参数对所有子命令都可用。没有它，`--data-dir` 只能出现在 `tasky --data-dir /tmp list`（在子命令之前）。有了它，`tasky list --data-dir /tmp`（在子命令之后）也能工作。

**`PathBuf`** — 标准库中用于表示文件系统路径的类型。clap 会自动把命令行字符串解析为 `PathBuf`——比用 `String` 再手动转换更安全。

### 传递给 storage

现在需要把 `data_dir` 传递给 `load_todos` 和 `save_todos`。修改 `storage.rs`：

```rust
use std::path::PathBuf;

pub fn data_file(custom_dir: Option<&PathBuf>) -> PathBuf {
    let dir = match custom_dir {
        Some(d) => d.clone(),
        None => dirs::config_dir()
            .expect("Cannot determine config directory")
            .join("tasky"),
    };
    fs::create_dir_all(&dir).expect("Cannot create config directory");
    dir.join("todos.json")
}

pub fn load_todos(custom_dir: Option<&PathBuf>) -> Result<Vec<Todo>> {
    let path = data_file(custom_dir);
    if !path.exists() {
        return Ok(Vec::new());
    }
    let content = fs::read_to_string(&path)
        .context("Failed to read todos file")?;
    let todos: Vec<Todo> = serde_json::from_str(&content)
        .unwrap_or_default();
    Ok(todos)
}

pub fn save_todos(todos: &[Todo], custom_dir: Option<&PathBuf>) -> Result<()> {
    let path = data_file(custom_dir);
    let json = serde_json::to_string_pretty(todos)
        .context("Failed to serialize todos")?;
    fs::write(&path, json)
        .context("Failed to write todos file")?;
    Ok(())
}
```

`data_file` 现在接受 `Option<&PathBuf>`：有值时用自定义目录，没有时用默认的 `dirs::config_dir()`。`load_todos` 和 `save_todos` 把这个参数透传进去。

### 更新 main.rs

`main` 函数需要把 `cli.data_dir` 传给所有 `load_todos` 和 `save_todos` 调用：

```rust
fn main() -> Result<()> {
    let cli = Cli::parse();
    let data_dir = cli.data_dir.as_ref();  // Option<&PathBuf>
    let mut todos = load_todos(data_dir)?;

    match cli.command {
        Commands::Add { content, priority, tags } => {
            cmd_add(&mut todos, content, priority, tags);
            save_todos(&todos, data_dir)?;
        }
        // ... 其他分支类似，save_todos 都加上 data_dir 参数
        Commands::List { all, priority, tag } => {
            cmd_list(&todos, all, priority, tag);
        }
        Commands::Clear => {
            cmd_clear(&mut todos);
            save_todos(&todos, data_dir)?;
        }
        // ...
    }

    Ok(())
}
```

`cli.data_dir.as_ref()` 把 `Option<PathBuf>` 转成 `Option<&PathBuf>`——借用引用而不是移动值，这样 `data_dir` 可以在多个 `save_todos` 调用中重复使用。

> [!HEADS-UP]
> 修改了 `load_todos` 和 `save_todos` 的签名后，`commands.rs` 中的 `cmd_stats` 不需要改——它只接收 `&[Todo]`，不直接调用 storage 函数。但如果你之前在其他地方调用了这两个函数（比如单元测试），记得同步更新调用点。

现在你的集成测试也可以用 `--data-dir` 指向临时目录，不再需要修改 `HOME` 环境变量：

```rust
#[test]
fn test_add_with_data_dir() {
    let tmp = tempfile::tempdir().unwrap();
    let dir = tmp.path().to_str().unwrap();

    Command::cargo_bin("tasky")
        .unwrap()
        .args(&["--data-dir", dir, "add", "测试"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Added #1"));
}
```

## 单元测试：验证内部逻辑

part 3 的练习 3 要求你给 `storage.rs` 添加单元测试。现在来完成它。

### 什么是单元测试

Rust 有两种测试：part 3 中你已经写过**集成测试**（在 `tests/` 目录下，测试整个程序的行为）。**单元测试**放在被测试的代码文件内部，验证单个函数或模块的逻辑。

单元测试放在文件底部的 `#[cfg(test)]` 模块中：

```rust
// storage.rs 的正式代码
pub fn load_todos() -> Result<Vec<Todo>> { ... }
pub fn save_todos(todos: &[Todo]) -> Result<()> { ... }

// 文件底部：单元测试
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn some_test() {
        // 测试代码
    }
}
```

三个新语法要素：

**`#[cfg(test)]`** — 条件编译属性。告诉编译器："这段代码只在 `cargo test` 时编译。" 正常运行 `cargo build` 或 `cargo run` 时，整个 `mod tests` 块被完全忽略——不编译、不占空间。

**`use super::*;`** — 把父模块（也就是 `storage.rs` 顶层）的所有公开和私有项导入测试模块。这让你可以直接调用 `load_todos()` 和 `save_todos()`，不需要写完整路径。`super::*` 中的 `super` 指向父模块，`*` 是 glob 导入（导入所有项）。

**`#[test]`** — 标记一个函数为测试用例。`cargo test` 会自动发现并运行所有带 `#[test]` 注解的函数。

### 断言宏

Rust 提供三个核心断言宏来验证测试结果：

```rust
// assert!(条件) —— 条件为 false 时测试失败
assert!(todos.len() > 0);

// assert_eq!(左, 右) —— 两者不相等时测试失败
assert_eq!(loaded.len(), 3);

// assert_ne!(左, 右) —— 两者相等时测试失败
assert_ne!(loaded.len(), 0);
```

断言失败时，Rust 会打印 panic 信息和失败位置。`assert_eq!` 和 `assert_ne!` 还会打印左右两边的实际值（要求类型实现 `Debug` + `PartialEq`），帮你快速定位问题。

### 给 storage.rs 添加测试

我们要验证的核心逻辑是"往返一致性"——写入一组 `Todo`，读回来，数据应该完全相同。

在 `storage.rs` 底部添加：

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use chrono::Local;

    #[test]
    fn save_and_load_roundtrip() {
        // 准备测试数据
        let todos = vec![
            Todo {
                id: 1,
                content: "测试任务一".to_string(),
                completed: false,
                created_at: Local::now(),
                priority: 0,
                completed_at: None,
                tags: vec!["test".to_string()],
            },
            Todo {
                id: 2,
                content: "测试任务二".to_string(),
                completed: true,
                created_at: Local::now(),
                priority: 2,
                completed_at: Some(Local::now()),
                tags: vec![],
            },
        ];

        // 写入
        save_todos(&todos, None).expect("save should succeed");

        // 读回
        let loaded = load_todos(None).expect("load should succeed");

        // 验证
        assert_eq!(loaded.len(), 2);
        assert_eq!(loaded[0].id, 1);
        assert_eq!(loaded[0].content, "测试任务一");
        assert_eq!(loaded[0].priority, 0);
        assert!(loaded[0].tags.contains(&"test".to_string()));
        assert_eq!(loaded[1].id, 2);
        assert!(loaded[1].completed);
        assert_eq!(loaded[1].priority, 2);
        assert!(loaded[1].completed_at.is_some());
    }

    #[test]
    fn load_default_does_not_crash() {
        // 冒烟测试：验证 load_todos(None) 能正常运行而不崩溃
        // 因为 load_todos(None) 指向默认路径，文件可能存在也可能不存在
        // （save_and_load_roundtrip 测试已在此前写入了数据文件）
        // 所以这里只验证函数能正常返回 Result，不检查具体内容
        let result = load_todos(None);
        assert!(result.is_ok());
    }
}
```

> [!ASIDE]
> **为什么不能写 `assert!(loaded.len() >= 0)`？** `Vec::len()` 返回 `usize`——无符号整数类型（在 64 位系统上等同于 `u64`）。无符号整数的取值范围是 `0..=18_446_744_073_709_551_615`，在定义上就不可能是负数。所以 `loaded.len() >= 0` 永远为 `true`，Rust 编译器的 `unused_comparisons` lint 会正确地警告这是一个无意义的比较。当你想断言集合为空时，用 `assert!(loaded.is_empty())` 或 `assert_eq!(loaded.len(), 0)`。

注意 `use super::*;` 把 `load_todos`、`save_todos`、`data_file` 等全部导入测试模块——包括 `data_file` 这个**私有**函数。这正是单元测试的一个优势：子模块可以访问父模块的私有项，让你能测试内部实现细节。

> [!HEADS-UP]
> 上面的 `save_and_load_roundtrip` 测试会写入**真实的** `todos.json` 文件（macOS 上位于 `~/Library/Application Support/tasky/todos.json`）。测试中传入 `None` 意味着使用默认存储路径——如果你当前有待办数据，运行测试会覆盖它。建议在测试前备份你的 `todos.json`。练习 5 会引导你给集成测试添加 `--data-dir` 支持，让测试使用独立的临时目录，彻底解决这个问题。

运行单元测试：

```bash
cargo test --bin tasky
```

`--bin tasky` 只运行 `tasky` 二进制目标中的单元测试（排除 `tests/` 目录下的集成测试）。预期输出：

```
running 2 tests
test tests::save_and_load_roundtrip ... ok
test tests::load_default_does_not_crash ... ok

test result: ok. 2 passed; 0 failed
```

如果你想同时运行所有测试（单元 + 集成），用 `cargo test -- --test-threads=1`。

> [!ASIDE]
> **测试私有函数。** 假设你在 `storage.rs` 中有一个私有的辅助函数 `fn validate_todos(todos: &[Todo]) -> bool`。因为 `tests` 是 `storage` 的子模块，它可以通过 `super::validate_todos(...)` 直接调用。这让你能测试内部逻辑，而不需要把它设为 `pub`——对外接口的最小化原则和可测试性并不矛盾。

## 自定义 trait 实现：fmt::Display

到目前为止，你的 `Todo` 类型使用了 `#[derive(Debug)]` 来支持 `{:?}` 格式化——输出长这样：`Todo { id: 1, content: "买菜", completed: false, ... }`。这对调试很有用，但用户不想看到这种格式。

`fmt::Display` trait 让你定义 `{}`（不带问号）的用户友好格式。实现 `Display` 后，你可以直接写 `println!("{}", todo)` 来打印格式化的待办信息。

### 实现 Display

在 `todo.rs` 中添加：

```rust
use std::fmt;

impl fmt::Display for Todo {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let status = if self.completed { "✓" } else { "○" };
        let priority_label = match self.priority {
            2 => "!!",
            1 => "!",
            _ => " ",
        };
        let tags_label = if self.tags.is_empty() {
            String::new()
        } else {
            format!(" [{}]", self.tags.join(", "))
        };
        write!(
            f,
            "[{}] {} {} {}{}",
            self.id,
            status,
            priority_label,
            self.content,
            tags_label,
        )
    }
}
```

几个新概念：

**`impl fmt::Display for Todo`** — 为 `Todo` 类型实现 `Display` trait。语法是 `impl TraitName for TypeName`。Rust 有一个"孤儿规则"（orphan rule）：你只能为**自己的类型**实现**外部的 trait**（或者为外部类型实现自己的 trait）。不能为两个都是外部的东西写实现——比如你不能 `impl Display for Vec<String>`。

**`fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result`** — 这是 `Display` trait 要求实现的方法签名。`&self` 是当前 `Todo` 实例的引用，`f` 是格式化器（你可以往里面写文本），返回值 `fmt::Result` 表示格式化是否成功。

**`write!(f, "...")`** — 这个宏把格式化字符串写入 `f`。用法和 `println!` 几乎一样，区别是它不输出到终端，而是写入你指定的 `Formatter`。它返回 `fmt::Result`，正好作为 `fmt` 方法的返回值。

实现 `Display` 后，你获得了两个额外好处：

```rust
// 1. 可以用 {} 打印
println!("{}", todo);  // [1] ○   买菜 [urgent, food]

// 2. 自动获得 .to_string()
let s = todo.to_string();  // "[1] ○   买菜 [urgent, food]"
```

第二个好处来自标准库的 **blanket implementation**（毯子实现）——它为所有实现了 `Display` 的类型自动实现了 `ToString` trait。你不需要手动写 `impl ToString for Todo`，编译器自动帮你做了。

> [!ASIDE]
> **`Display` vs `Debug`。** `Debug`（用 `{:?}` 打印）是给开发者看的——详细的结构信息，用于调试。`Display`（用 `{}` 打印）是给用户看的——简洁友好的展示。`Debug` 可以用 `#[derive(Debug)]` 自动派生，`Display` 必须手写实现，因为编译器不知道你想怎么"展示"一个类型。你在 part 1 中给 `Todo` 加过 `#[derive(Debug)]`——那个是自动的。现在手写的 `Display` 实现让你完全控制输出格式。

### 用 Display 简化 print_todos

现在 `Todo` 有了 `Display`，你可以用 `{}` 替代 `print_todos` 中的一些手动格式化。不过注意：`print_todos` 还使用了 `colored` 的着色功能（`.green()`、`.strikethrough()` 等），这些在 `Display` 的 `write!` 宏中不能直接用——`write!` 只做纯文本格式化，不支持 ANSI 颜色。所以 `print_todos` 仍然需要保留着色逻辑，`Display` 适合作为纯文本展示（比如导出到文件、日志记录）的补充。

## 检查点

> [!PREDICT]
> 运行之前想一想：`--data-dir` 是全局参数。运行 `tasky add --data-dir /tmp/test "新任务"`（`--data-dir` 在子命令**之后**）能正常工作吗？

**运行以下命令验证你目前的工作：**

```bash
cargo build
```

预期输出：编译成功，无错误。

```bash
cargo run -- stats
```

预期输出：显示当前的待办统计信息。

```bash
cargo run -- add "测试 clear 命令" -p 1 --tags test
```

预期输出：

```
✔ test Added #N: 测试 clear 命令   [high]
```

```bash
cargo run -- done N
```

（把 `N` 替换为上一步添加的 ID）

预期输出：

```
✔ Done #N: 测试 clear 命令
```

```bash
cargo run -- list --all
```

预期输出：能看到已完成的待办。

```bash
cargo run -- clear
```

预期输出：

```
✓ Cleared 1 completed todo(s)
```

```bash
cargo run -- list --all
```

预期输出：刚才标记完成的那条待办已消失。

```bash
cargo run -- --data-dir /tmp/tasky-test add "临时目录测试"
```

预期输出：

```
✔ Added #1: 临时目录测试
```

确认文件写到了 `/tmp/tasky-test/todos.json`：

```bash
cat /tmp/tasky-test/todos.json
```

```bash
cargo test --bin tasky
```

预期输出：单元测试通过。

```bash
cargo test -- --test-threads=1
```

预期输出：所有测试（单元 + 集成）通过。

**可能的错误：**

- 如果看到 `method not found in 'Cli'`（关于 `parse`），你可能在 `main.rs` 中忘了 `use clap::Parser;`——`parse()` 是 `Parser` trait 的方法，需要 trait 在当前作用域中。
- 如果看到 `this function takes 2 arguments but 1 argument was supplied`（关于 `save_todos`），你可能忘了更新某个调用点——修改函数签名后，所有调用 `save_todos` 的地方都需要加上第二个参数。
- 如果看到 `cannot borrow 'cli.data_dir' as immutable`（在 `match` 中），你可能在 `match cli.command` 消耗了 `cli` 之后还想访问 `cli.data_dir`——解决方法是在 `match` 之前先用 `let data_dir = cli.data_dir.as_ref();` 取出引用。
- 如果看到 `expected PathBuf, found &PathBuf`（关于 `data_file` 的参数），你可能在传递 `data_dir` 时没有用 `.as_ref()` 转换——`Option<PathBuf>` 和 `Option<&PathBuf>` 是不同类型。

## 接下来

你的 `tasky` 现在功能齐全：七个子命令、全局参数、单元测试、集成测试、模块化的代码结构。四个部分覆盖的 Rust 基础概念——所有权、借用、`match`、迭代器、`Option`/`Result`、trait、模块系统——构成了 Rust 编程的核心骨架。

从这里出发，你可以朝多个方向继续深入：用 `thiserror` crate 定义自定义错误类型（比 `anyhow` 更适合库 crate），用 `tokio` 引入异步编程（让 `tasky` 支持网络同步），或者用 `ratatui` 构建终端交互界面（TUI）。每个方向都会复用你在四个部分中建立的基础知识。

在继续之前，用你自己的话回答：`Vec::retain()` 和 `.filter().collect()` 都能实现"只保留符合条件的元素"，它们的区别是什么？在什么场景下你会选择 `retain` 而不是 `filter`？

## 练习

1. **给 `Todo` 实现 `PartialEq`。** 添加 `#[derive(PartialEq)]` 到 `Todo` 结构体。然后在单元测试中用 `assert_eq!` 比较两个内容相同的 `Todo` 是否相等。`PartialEq` 让你的类型支持 `==` 和 `!=` 运算符——`assert_eq!` 依赖它来比较值。

2. **给 `list` 命令添加 `--sort` 选项。** 支持按 `id`（默认）、`priority`（从高到低）、`created`（按创建时间）排序。在 `cli.rs` 中给 `List` 变体添加 `#[arg(short, long, default_value = "id")] sort: String`。在 `cmd_list` 中用 `items.sort_by(|a, b| ...)` 排序。提示：`sort_by` 接受一个闭包，返回 `std::cmp::Ordering`（`Less`、`Equal`、`Greater`）。

3. **给 `Todo` 实现 `From<&str>`。** 让用户可以用 `Todo::from("买菜")` 快速创建一个简单的待办（ID 为 0、优先级为 0、无标签）。语法是 `impl From<&str> for Todo { fn from(s: &str) -> Self { ... } }`。实现 `From` 后，你自动获得 `Into`（`let todo: Todo = "买菜".into();`）——这是标准库的另一个 blanket implementation。

4. **改进 `cmd_stats` 的展示。** 给 `stats` 命令添加一个"完成率"百分比（如 `完成率: 60%`），以及一个"最旧未完成待办"显示（创建时间最早且未完成的待办的内容和已等待天数）。使用 `chrono` 的 `Local::now().signed_duration_since(created_at)` 计算时间差。

5. **给集成测试加上 `--data-dir`。** 把 `tests/cli_test.rs` 中所有测试改为使用 `--data-dir` 参数指向临时目录，替代修改 `HOME` 环境变量的方式。完成后你的测试不再需要 `unsafe { set_var(...) }`，也不再需要 `--test-threads=1`（因为每个测试使用独立的临时目录，互不干扰）。

## 来源

1. [The Rust Programming Language — Chapter 10: Traits](https://doc.rust-lang.org/book/ch10-02-traits.html) — trait 的定义、实现和孤儿规则的官方教程。
2. [std::fmt::Display documentation](https://doc.rust-lang.org/std/fmt/trait.Display.html) — `Display` trait 的方法签名、`write!` 宏用法和实现示例。
3. [The Rust Programming Language — Chapter 11: Writing Tests](https://doc.rust-lang.org/book/ch11-01-writing-tests.html) — `#[test]`、`#[cfg(test)]`、断言宏和测试组织。
4. [clap derive documentation](https://docs.rs/clap/latest/clap/_derive/index.html) — 全局参数（`global = true`）、`Cli` 结构体字段和子命令的 derive 语法。
5. [The Rust Programming Language — Trait Bounds](https://doc.rust-lang.org/book/ch10-02-traits.html#trait-bound-syntax) — trait bound 语法、`where` 子句和 blanket implementation。
6. [std::vec::Vec::retain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.retain) — `retain` 方法的文档，与 `filter` 的区别。
