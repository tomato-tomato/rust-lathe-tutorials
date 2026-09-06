# 用 Rust 构建命令行待办事项应用

你读过不少 Rust 教程了。它们大多数让你在终端打印 `"Hello, world!"`，然后就此止步。有些让你写个猜数字游戏。但你真正想做的，是用 Rust 构建一个**每天都在用的工具**——一个在终端里管理待办事项的 CLI 应用，带子命令、JSON 持久化、彩色输出。

这篇教程会给你这些。我们会构建一个叫 `tasky` 的命令行工具，你用它管理自己真正的待办事项。在这个过程中，你会碰到 Rust 几乎所有核心概念：所有权、借用、枚举模式匹配、trait 派生、错误处理、文件 I/O 和序列化。每一个都不是孤立出现的——它们在你构建一个真实工具时自然浮现。

本部分结束时，你将拥有一个可以运行的 `tasky`：能添加任务、列出未完成项、标记完成、删除条目，所有数据保存在磁盘上，关掉终端再打开依然在那里。

## 你将构建什么

一个名为 `tasky` 的命令行待办事项管理器。它支持四个子命令——`add`、`list`、`done`、`remove`——用 JSON 文件持久化数据，用 ANSI 转义码给终端输出上色。最终产物是一个你编译一次、用几年的小工具。

```
$ tasky add "读完这篇教程"
✓ Added #1: 读完这篇教程

$ tasky add "配置 Rust 开发环境"
✓ Added #2: 配置 Rust 开发环境

$ tasky done 1
✓ Done #1: 读完这篇教程

$ tasky list
  Pending (1):
    [2] 配置 Rust 开发环境  (2026-06-13 14:30)

$ tasky list --all
  All (2):
    [1] 读完这篇教程  ✓ done  (2026-06-13 14:30)
    [2] 配置 Rust 开发环境  (2026-06-13 14:32)
```

## 前置条件

- **Rust 1.96.0** 或更高版本（通过 [rustup](https://rustup.rs/) 安装）
- 了解终端基本操作（`cd`、运行命令）
- 不需要 Rust 经验——我们从零开始

## 一个空的 Cargo 项目

打开终端，创建项目：

```bash
cargo new tasky
cd tasky
```

Cargo 生成了一个最小骨架：`Cargo.toml` 和 `src/main.rs`。打开 `Cargo.toml`，确认 `edition` 字段是 `"2024"`（`cargo new` 在 Rust 1.85+ 上会默认生成这个值），然后在文件底部添加五个依赖项：

```toml
[package]
name = "tasky"
version = "0.1.0"
edition = "2024"

[dependencies]
clap = { version = "4", features = ["derive"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = "0.4"
dirs = "5"
```

> [!ASIDE]
> **`edition` 是什么？** Rust 的 edition 是一个语言版本开关，控制某些语法的含义和编译器的默认行为。它与编译器版本（`rustc --version`）是独立的——你可以用 Rust 1.96.0 编译 `edition = "2021"` 的项目，也可以用旧编译器编译 `edition = "2024"` 的项目（只要编译器版本不低于该 edition 的最低要求）。edition 之间高度兼容，2024 只在少数边缘场景（如 `gen` 保留关键字、`unsafe extern` 块）引入了不兼容变化。对于本教程的代码，`edition = "2021"` 和 `"2024"` 的行为完全相同。

这五个 crate 的选择不是随意的。clap 是 Rust CLI 生态的事实标准，其 [derive API](https://docs.rs/clap/latest/clap/_derive/index.html) 让你用结构体和枚举定义命令行界面，编译器帮你生成参数解析代码。serde + serde_json 负责把 `Todo` 结构体变成 JSON 文本再变回来——[序列化](https://docs.rs/serde/latest/serde/)和反序列化是 Rust 处理结构化数据的核心模式。chrono 处理时间戳。dirs 提供跨平台的配置目录路径——不再硬编码 `~/.config`。

> [!DESIGN-NOTE]
> **为什么不手动解析 `std::env::args()`？**
>
> 对于一个只有四个命令的工具，用 `match args[1].as_str()` 完全可行。但手动解析很快会失控：你需要自己处理 `--help`、`-h`、版本号、无效输入的错误提示。clap 的 derive API 把这些全部替你做了——当你给 `list` 子命令添加一个 `--all` 标志时，clap 自动在 `--help` 输出中列出它，自动验证用户输入的类型，自动处理缺失参数。手动解析是可行的，但 clap 给你的是工业级的 CLI 体验，成本只是几行 derive 注解。

## 一条待办事项的两个面孔

每个待办事项在内存中是一个结构体，在磁盘上是一段 JSON 文本。`serde` 的 derive 宏在两者之间搭桥。在 `src/main.rs` 的顶部，添加导入和结构体定义：

```rust
use clap::{Parser, Subcommand};
use serde::{Deserialize, Serialize};
use std::fs;
use std::path::PathBuf;

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Todo {
    id: u32,
    content: String,
    completed: bool,
    created_at: String,
}
```

五个 derive 宏，各司其职：`Debug` 让你用 `{:?}` 打印结构体；`Clone` 让你可以复制一个 `Todo`（稍后过滤列表时有用）；`Serialize` 和 `Deserialize` 是 serde 的核心——它们让你的结构体能自动转换为 JSON 和从 JSON 还原。编译器在编译时检查：结构体的每个字段都必须实现了这两个 trait，否则报错。这就是 Rust 类型系统保护你的方式——不存在"运行时发现序列化失败"的情况。

> [!ASIDE]
> `created_at` 为什么是 `String` 而不是 chrono 的 `DateTime<Local>`？因为这是一个初学者的第一个 Rust 项目，`DateTime<Local>` 的泛型参数和 trait bound 会在你还没准备好理解它们的时候引入噪音。用 `String` 存储格式化的时间字符串是一个实用主义的妥协——它简单、可读、序列化免费。等你对 Rust 更熟悉后，练习 3 会引导你把它换成类型安全的 `DateTime`。

在结构体下方添加构造函数。`impl` 块是 Rust 给类型添加方法的地方：

```rust
impl Todo {
    fn new(id: u32, content: String) -> Self {
        let now = chrono::Local::now().format("%Y-%m-%d %H:%M").to_string();
        Todo {
            id,
            content,
            completed: false,
            created_at: now,
        }
    }
}
```

[`chrono::Local::now()`](https://docs.rs/chrono/latest/chrono/struct.Local.html#method.now) 返回当前本地时间，`.format("%Y-%m-%d %H:%M")` 生成形如 `"2026-06-13 14:30"` 的字符串。`Self` 是 `Todo` 的别名——在 `impl Todo` 块里，写 `Self` 比写 `Todo` 更清晰。

注意 `content: String` 而非 `content: &str`。`String` 拥有自己的数据（堆上分配），`&str` 是一个借用引用。对于需要长期持有的数据——比如存储在 `Vec<Todo>` 里的任务文本——你需要 `String`。这是 Rust 所有权模型的第一课：数据的所有者负责它的生命周期，`Vec<Todo>` 拥有每个 `Todo`，每个 `Todo` 拥有自己的 `content`。

## 四个命令，一个枚举

clap 的 derive API 用两样东西定义你的 CLI 界面：一个带 `#[derive(Parser)]` 的结构体做入口，一个带 `#[derive(Subcommand)]` 的枚举做命令分支。在 `Todo` 的 `impl` 块之后添加：

```rust
#[derive(Parser)]
#[command(name = "tasky", version, about = "A tiny todo manager")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Add a new todo
    Add {
        /// The task description
        #[arg(required = true)]
        content: String,
    },
    /// List todos (pending by default)
    List {
        /// Show all todos including completed
        #[arg(short, long)]
        all: bool,
    },
    /// Mark a todo as done
    Done {
        /// The ID of the todo to complete
        id: u32,
    },
    /// Remove a todo
    Remove {
        /// The ID of the todo to remove
        id: u32,
    },
}
```

这段代码的密度值得逐行拆解。

`#[derive(Parser)]` 告诉 clap：`Cli` 结构体是整个命令行的根。`#[command(name = "tasky", version, about = "...")]` 设置元数据——当用户运行 `tasky --version` 或 `tasky --help` 时，clap 用这些值填充输出。`version` 不加 `= "..."` 时，clap 自动从 `Cargo.toml` 的 `version` 字段读取。

`#[command(subcommand)]` 标记 `command` 字段：它的值不是位置参数，而是一个子命令。clap 看到这个注解后，把 `Commands` 枚举的每个变体变成一个 CLI 子命令。

`///` 三斜杠注释不只是文档——clap 读取它们作为 `--help` 输出的描述文本。运行 `tasky add --help` 时，`/// Add a new todo` 出现在输出中。这是 derive API 最贴心的设计之一：你写的注释同时服务人类和机器。

`#[arg(short, long)]` 给 `all` 字段生成 `-a` 短标志和 `--all` 长标志。不带这个注解的字段——比如 `Add` 的 `content` 和 `Done` 的 `id`——自动成为位置参数。clap 根据 Rust 类型推断解析方式：`String` 解析为文本，`u32` 解析为非负整数（输入 `-5` 或 `abc` 时自动报错），`bool` 自动变成开关标志。

> [!HEADS-UP]
> 变体名决定子命令名，而 clap 默认将其转为 kebab-case。`Add` 变成 `add`，`Remove` 变成 `remove`。如果你有一个叫 `MarkDone` 的变体，它会变成 `mark-done`。这个转换是自动的，不需要你操心——但如果你想要特定的名称，可以用 `#[command(name = "...")]` 覆盖。

现在你的 `Cli` 和 `Commands` 已经定义好了，但 `main()` 函数还是 `cargo new` 生成的默认版本——`fn main() { println!("Hello, world!"); }`。clap 的 derive 宏生成了所有解析代码，但如果没人调用 `Cli::parse()`，这些代码就是死代码，编译器会报一堆 `unused` 警告。

先把 `main()` 替换成一个最小版本，让 clap 接管命令行：

```rust
fn main() {
    let _cli = Cli::parse();
}
```

`Cli::parse()` 读取命令行参数，返回填充好的 `Cli` 结构体。下划线前缀 `_cli` 告诉编译器"我知道这个变量暂时没被使用"——因为我们还没有命令路由逻辑，这里只是验证 CLI 界面能正常工作。在本部分后面的"命令路由和终端着色"章节，你会把这个临时 `main` 替换成完整版本。

运行一下看看效果：

```bash
cargo run -- --help
```

你应该看到：

```
A tiny todo manager

Usage: tasky <COMMAND>

Commands:
  add     Add a new todo
  list    List todos (pending by default)
  done    Mark a todo as done
  remove  Remove a todo
  help    Print this message or the help for the given subcommand(s)

Options:
  -h, --help     Print help
  -v, --version  Print version
```

不需要你写任何 `--help` 的处理代码。clap 从你的结构体和注释自动生成了这一切。

> [!NOTE]
> 编译时你可能还会看到一些 `unused import: std::fs` 或 `struct Todo is never constructed` 之类的警告。这些是预期行为——文件顶部导入了后面章节才用到的模块，`Todo` 也还没有被任何逻辑构造。等下一节"一个知道存在哪里的存储层"添加完存储函数，这些警告会自然消失。

## 一个知道存在哪里的存储层

待办数据需要一个家。硬编码路径 `"todos.json"` 会把文件丢在用户运行命令的当前目录下——今天在公司跑一次，明天在家跑一次，数据就分裂了。

`dirs` crate 提供符合操作系统惯例的配置目录：macOS 上是 `~/Library/Application Support/`，Linux 上是 `~/.config/`（遵循 XDG 规范），Windows 上是 `%APPDATA%`。在结构体和枚举定义之后添加两个函数：

```rust
fn data_file() -> PathBuf {
    let dir = dirs::config_dir()
        .expect("Cannot determine config directory")
        .join("tasky");
    fs::create_dir_all(&dir).expect("Cannot create config directory");
    dir.join("todos.json")
}

fn load_todos() -> Vec<Todo> {
    let path = data_file();
    if !path.exists() {
        return Vec::new();
    }
    let content = fs::read_to_string(&path)
        .expect("Failed to read todos file");
    serde_json::from_str(&content).unwrap_or_default()
}
```

[`dirs::config_dir()`](https://docs.rs/dirs/latest/dirs/fn.config_dir.html) 返回 `Option<PathBuf>`——在极少数情况下（比如没有家目录的系统环境），它返回 `None`。我们用 `.expect()` 在 `None` 时 panic 并打印一条人类可读的消息。在真实的发布级应用中，你会用 `anyhow` 或 `thiserror` crate 做更优雅的错误处理。但对于一个个人工具，`expect()` 够用，而且它教你一个重要的 Rust 模式：`Option` 和 `Result` 必须被显式处理，编译器不允许你忽略它们。

`load_todos()` 处理了三种情况：文件不存在（返回空列表）、文件存在且 JSON 合法（反序列化为 `Vec<Todo>`）、文件存在但 JSON 损坏（`unwrap_or_default()` 返回空的 `Vec<Todo>`，不会 panic）。第三种情况是故意的——你的工具不应该因为一次磁盘写入失败就再也打不开。

> [!ASIDE]
> `unwrap_or_default()` 是一个你可能在很多地方遇到的方法。它做两件事：如果 `Result` 是 `Ok(value)`，返回 `value`；如果是 `Err(_)`，返回该类型的 `Default` 实现。`Vec<Todo>` 的 `Default` 是空的 `Vec`。这种"出错就静默回退到空状态"的策略适合这里——但如果你想在 JSON 损坏时警告用户，把 `unwrap_or_default()` 换成显式的 `match` 就行。

现在写保存函数。在 `load_todos` 之后添加：

```rust
fn save_todos(todos: &[Todo]) {
    let path = data_file();
    let json = serde_json::to_string_pretty(todos)
        .expect("Failed to serialize todos");
    fs::write(&path, json).expect("Failed to write todos file");
}
```

参数类型是 `&[Todo]`——一个对 `Todo` 切片的借用引用。这意味着 `save_todos` 不获取数据的所有权，它只是读。`serde_json::to_string_pretty()` 把 `&[Todo]` 序列化为带缩进的 JSON 字符串，方便你用文本编辑器直接查看或编辑 `todos.json`。`fs::write()` 原子地写入整个文件——如果写入失败（磁盘满了、权限不够），错误会被 `.expect()` 捕获。

## 命令路由和终端着色

数据层就绪，CLI 界面就绪。现在需要把它们连起来。在 `save_todos` 函数之后，添加四个命令处理函数，然后写 `main`：

```rust
fn cmd_add(todos: &mut Vec<Todo>, content: String) {
    let new_id = todos.iter().map(|t| t.id).max().unwrap_or(0) + 1;
    let todo = Todo::new(new_id, content);
    println!("\x1b[32m✓\x1b[0m Added #{}: {}", todo.id, todo.content);
    todos.push(todo);
}

fn cmd_list(todos: &[Todo], all: bool) {
    let items: Vec<&Todo> = if all {
        todos.iter().collect()
    } else {
        todos.iter().filter(|t| !t.completed).collect()
    };

    if items.is_empty() {
        println!("Nothing here. Use `tasky add <text>` to create one.");
        return;
    }

    let label = if all { "All" } else { "Pending" };
    println!("  {} ({}):", label, items.len());
    for t in &items {
        if t.completed {
            println!(
                "    \x1b[9m[{}] {}\x1b[0m  \x1b[32m✓ done\x1b[0m  ({})",
                t.id, t.content, t.created_at
            );
        } else {
            println!("    [{}] {}  ({})", t.id, t.content, t.created_at);
        }
    }
}

fn cmd_done(todos: &mut Vec<Todo>, id: u32) {
    match todos.iter_mut().find(|t| t.id == id) {
        Some(todo) => {
            todo.completed = true;
            println!("\x1b[32m✓\x1b[0m Done #{}: {}", id, todo.content);
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
```

四个函数，四种与 `Vec<Todo>` 交互的方式。注意参数签名的区别：`cmd_add` 和 `cmd_done` 需要 `&mut Vec<Todo>`（它们修改列表），`cmd_list` 只需要 `&[Todo]`（只读）。`cmd_remove` 使用 `&mut` 因为 `retain()` 就地删除元素。Rust 的借用检查器在编译时确保：你不可能同时拥有一个可变引用和一个不可变引用。这意味着在你调用 `cmd_list(todos, true)` 的时候，编译器保证没有人正在修改 `todos`。

> [!ASIDE]
> `cmd_remove` 用了一个小技巧来检测删除是否成功：比较 `retain()` 前后的长度。另一种方式是先 `find()` 再 `retain()`，但那样要遍历两次。还有一种更 Rust 的方式是用 `position()` 找到索引然后 `swap_remove()`——但 `retain()` 对初学者最直观，而且对几百条待办来说性能没有区别。

终端着色用的是 ANSI 转义序列：`\x1b[32m` 切换到绿色（用于 `✓` 标记），`\x1b[9m` 是删除线（用于已完成的任务），`\x1b[0m` 重置所有样式。这些转义序列在几乎所有现代终端中都能工作——macOS Terminal、iTerm2、GNOME Terminal、Windows Terminal。唯一的例外是旧版 Windows `cmd.exe`，但那是个我们正在告别的世界。

最后，`main` 函数把所有零件组装在一起。在所有函数定义之后，把之前的临时 `main`（只有 `let _cli = Cli::parse();` 那个）替换成完整版本：

```rust
fn main() {
    let cli = Cli::parse();
    let mut todos = load_todos();

    match cli.command {
        Commands::Add { content } => {
            cmd_add(&mut todos, content);
            save_todos(&todos);
        }
        Commands::List { all } => {
            cmd_list(&todos, all);
        }
        Commands::Done { id } => {
            cmd_done(&mut todos, id);
            save_todos(&todos);
        }
        Commands::Remove { id } => {
            cmd_remove(&mut todos, id);
            save_todos(&todos);
        }
    }
}
```

`Cli::parse()` 是 clap derive API 的入口——它读取 `std::env::args()`，解析命令行参数，返回填充好的 `Cli` 结构体。如果用户输入无效（比如忘了必填参数），`parse()` 会自动打印错误信息并退出程序，你不需要写任何错误处理代码。

`match cli.command` 对 `Commands` 枚举做穷举匹配——编译器保证你处理了所有四个变体。如果你后来添加了第五个子命令但忘了在 `match` 里处理它，编译会直接失败。这是 Rust 枚举最强大的特性之一：新增变体时，编译器会在每个遗漏的 `match` 处报错，让你不可能"忘记处理某个情况"。

`List` 分支不调用 `save_todos()`——列出任务不应该修改数据文件。这个看似微小的设计选择其实很重要：如果 `list` 也写文件，两个并发运行的 `tasky list` 可能互相覆盖数据。只读操作不写文件，这是一个值得养成的习惯。

## 检查点

> [!PREDICT]
> 在运行之前想一想：`cargo build` 编译整个项目时，编译器会检查哪些东西？如果你把 `Commands::Done { id }` 分支从 `match` 里删掉，编译会发生什么？

**运行以下命令验证你目前的工作：**

```bash
cargo build
```

预期输出：

```
   Compiling proc-macro2 v1.x.x
   Compiling unicode-ident v1.x.x
   ...（编译依赖项）
   Compiling tasky v0.1.0 (/path/to/tasky)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in Xs
```

```bash
cargo run -- add "学习 Rust 所有权"
```

预期输出：

```
✓ Added #1: 学习 Rust 所有权
```

```bash
cargo run -- add "理解借用检查器"
```

预期输出：

```
✓ Added #2: 理解借用检查器
```

```bash
cargo run -- list
```

预期输出：

```
  Pending (2):
    [1] 学习 Rust 所有权  (2026-06-13 14:30)
    [2] 理解借用检查器  (2026-06-13 14:30)
```

```bash
cargo run -- done 1
```

预期输出：

```
✓ Done #1: 学习 Rust 所有权
```

```bash
cargo run -- list
```

预期输出（只剩未完成的）：

```
  Pending (1):
    [2] 理解借用检查器  (2026-06-13 14:30)
```

```bash
cargo run -- list --all
```

预期输出（显示全部，已完成的带删除线）：

```
  All (2):
    [1] 学习 Rust 所有权  ✓ done  (2026-06-13 14:30)
    [2] 理解借用检查器  (2026-06-13 14:30)
```

```bash
cargo run -- remove 1
```

预期输出：

```
Removed #1
```

```bash
cargo run -- list --all
```

预期输出（只剩一条）：

```
  All (1):
    [2] 理解借用检查器  (2026-06-13 14:30)
```

**可能的错误：**

- 如果看到 `cannot find value 'Commands' in this scope`，你可能把 `Commands` 枚举放在了 `use clap::Subcommand;` 之前——确保 `use` 语句在文件顶部。
- 如果看到 `the trait bound 'Commands: Subcommand' is not satisfied`，检查 `#[derive(Subcommand)]` 是否在 `enum Commands` 上方。
- 如果看到 `thread 'main' panicked at 'Cannot determine config directory'`，你的系统环境可能缺少家目录配置（这在容器环境中常见）。可以在代码中用 `unwrap_or_else(|| PathBuf::from("."))` 替换 `.expect()` 作为临时回退。

## 接下来

你已经拥有了一个可工作的 CLI 待办工具——但它是"能用"阶段，还不是"好用"阶段。后续部分将回答这些问题：如何用 `anyhow` 和 `thiserror` 把 `expect()` 替换为优雅的错误处理？如何给 `list` 添加优先级过滤和搜索？如何用 `colored` crate 替代裸的 ANSI 转义序列让着色代码更可维护？如何编写集成测试验证 CLI 的行为？如何发布到 crates.io 让全世界用 `cargo install tasky` 安装？

在继续之前：用两句话解释——为什么 `match cli.command` 中的穷举检查在添加新子命令时特别有价值？如果你改用 `if-else` 链来分发命令，会失去什么保护？用自己的话写出来，让一个没读过这篇教程的 Rust 初学者也能理解。

## 练习

1. **给 `add` 命令添加 `--priority` 选项。** 在 `Todo` 结构体中加一个 `priority: u8` 字段（0 = 普通，1 = 高，2 = 紧急），在 `Add` 变体中添加 `#[arg(short, long, default_value_t = 0)] priority: u8`，在 `cmd_add` 中传递它，在 `cmd_list` 中根据优先级用不同颜色显示。

2. **添加 `search` 子命令。** 接受一个关键词字符串，列出所有 `content` 包含该关键词的待办（用 `content.contains(&keyword)` 匹配）。提示：在 `Commands` 枚举中添加新变体，然后在 `main` 的 `match` 中处理它——注意编译器会强制你处理这个新分支。

3. **把 `created_at` 从 `String` 改为 `chrono::DateTime<chrono::Local>`。** 你需要在 `Cargo.toml` 的 `chrono` 依赖中添加 `features = ["serde"]`。观察编译器如何引导你修改构造函数和序列化代码——这就是 Rust 的类型系统在帮你做重构。

4. **让 `done` 命令支持 `--undo` 标志。** 如果用户运行 `tasky done --undo 3`，把 ID 为 3 的待办标记为未完成。在 `Commands::Done` 变体中添加 `#[arg(long)] undo: bool` 字段，在 `cmd_done` 中根据这个标志切换 `completed` 状态。

5. **查看 `~/.config/tasky/todos.json`（macOS 上是 `~/Library/Application Support/tasky/todos.json`）的内容。** 手动编辑它——添加一条新的 JSON 对象，然后运行 `tasky list` 验证它被正确读取。如果 JSON 格式损坏了会发生什么？

## 来源

1. [clap derive API reference](https://docs.rs/clap/latest/clap/_derive/index.html) —— derive 宏的完整属性文档，本教程 CLI 界面的定义基础。
2. [serde documentation](https://serde.rs/) —— 序列化和反序列化框架的设计理念和 API 参考。
3. [chrono API reference](https://docs.rs/chrono/latest/chrono/) —— 时间处理库，本教程使用 `Local::now()` 和 `.format()` 生成时间戳。
4. [Rust CLI Book](https://rust-cli.github.io/book/) —— 官方 Rust CLI 开发指南，涵盖项目结构、错误处理和测试最佳实践。
5. [dirs crate documentation](https://docs.rs/dirs/latest/dirs/) —— 跨平台配置/数据/缓存目录路径解析。
