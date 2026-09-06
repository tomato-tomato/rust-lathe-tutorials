# 用 Rust 构建命令行待办事项应用 — 第二部分

> [!RECALL]
> 继续之前快速回忆：在 part 1 中，`cmd_add` 和 `cmd_done` 的参数是 `&mut Vec<Todo>`，而 `cmd_list` 的参数是 `&[Todo]`。这个区别意味着什么？如果你在 `cmd_list` 中尝试调用 `todos.push(...)`，编译器会报什么错？

在第一部分，你拥有了一个能运行的 `tasky`：四个子命令、JSON 持久化、ANSI 着色。但代码里有几个粗糙的地方——`println!` 中散落着 `\x1b[32m` 这样的魔法字符串，`.expect()` 在任何 I/O 失败时直接 panic，`cmd_list` 和 `cmd_search`（如果你做了练习 2 的话）共享几乎相同的打印逻辑。

本部分会把这些粗糙的边缘打磨掉。你会给待办加上优先级字段、实现关键词搜索、用 `colored` crate 替换裸 ANSI 转义序列、用 `anyhow` 替换 `.expect()` 让错误处理更优雅。每一步都是 part 1 中已有概念的自然延伸——`match` 穷举匹配、迭代器链、借用引用——你会在不同的上下文中反复遇到它们，直到它们变成肌肉记忆。

本部分结束时，你的 `tasky` 将支持优先级标记、关键词搜索、可维护的终端着色、和友好的错误报告——不再 panic。

## 你将构建什么

在 part 1 的基础上，给 `tasky` 添加三个改进：`--priority` 标志让每条待办有紧急程度、`search` 子命令按关键词过滤、`anyhow` 错误处理让程序在文件损坏时告诉你"发生了什么"而不是直接崩溃。终端输出改用 `colored` crate 着色——告别 `\x1b[32m`，拥抱 `.green()`。

```
$ tasky add -p 2 "修复生产环境 bug"
✓ Added #3: 修复生产环境 bug  [urgent]

$ tasky search "Rust"
  Search: "Rust" (2):
    [1] 学习 Rust 所有权  (2026-06-13 14:30)
    [2] 理解借用检查器  (2026-06-13 14:32)

$ tasky list
  Pending (3):
    [1] 学习 Rust 所有权  (2026-06-13 14:30)
    [2] 理解借用检查器  (2026-06-13 14:32)
    [3] 修复生产环境 bug  !! urgent  (2026-06-14 09:15)
```

## 前置条件

- part 1 完成的代码（能通过检查点的 `tasky` 项目）
- 如果你做了 part 1 的练习 1（优先级）或练习 2（搜索），先把那些改动**还原**——本部分会从头实现它们，这次有更详细的解释

打开 `Cargo.toml`，添加两个新依赖项：

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = "0.4"
dirs = "5"
colored = "2"
anyhow = "1"
```

`colored` 提供链式着色 API（`.green()`、`.red()`、`.strikethrough()`），`anyhow` 提供应用级错误处理（`Result` 类型别名、`?` 操作符、上下文标注）。两者都是 Rust CLI 生态中最常用的 crate。

## 一个共享的打印函数

part 1 的 `cmd_list` 里有一段 `for` 循环，根据 `completed` 状态用不同的 ANSI 转义码打印待办。如果你做了练习 2 的 `cmd_search`，你几乎复制粘贴了同一段代码。这是代码重复的信号——提取一个共享函数。

在 `use` 语句区域，把 `use std::fs;` 和 `use std::path::PathBuf;` 旁边加上 colored 的导入：

```rust
use colored::Colorize;
```

[`Colorize`](https://docs.rs/colored/latest/colored/trait.Colorize.html) 是一个 trait——Rust 中定义行为共享接口的机制。导入它之后，所有 `&str` 和 `String` 都自动获得 `.green()`、`.red()`、`.bold()`、`.strikethrough()` 等方法。你不需要修改任何类型，trait 的 blanket implementation 帮你做了。

现在提取打印逻辑。在所有 `cmd_*` 函数**之前**添加：

```rust
fn print_todos(label: &str, items: &[&Todo]) {
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
            println!(
                "    {} {}  {}  ({}){}",
                format!("[{}]", t.id).dimmed(),
                t.content.strikethrough(),
                "✓ done".green(),
                t.created_at,
                priority_tag,
            );
        } else {
            println!(
                "    [{}] {}  ({}){}",
                t.id, t.content, t.created_at, priority_tag,
            );
        }
    }
}
```

这段代码里有几个值得注意的新模式，逐个拆解。

**参数 `&[&Todo]`——引用的切片。** 这个类型读起来有点绕：它是一个切片（`[...]`），切片里每个元素是一个 `Todo` 的借用引用（`&Todo`），整个切片本身也是借用的（`&`）。为什么不是 `&[Todo]`？因为 `cmd_list` 和 `cmd_search` 都通过 `.iter().filter(...).collect()` 生成一个 `Vec<&Todo>`（引用集合），传给 `&[&Todo]` 参数是自然匹配。如果你传 `&[Todo]`，则需要先把 `Todo` 克隆一份——对打印来说完全没必要。

**`match t.priority`——优先级标签。** 这是 part 1 中 `match cli.command` 的同一个模式，只是用在更小的枚举场景上。`2` 匹配紧急、`1` 匹配高优先级、`_` 通配其他所有值。`match` 在 Rust 中无处不在——只要你需要根据一个值的不同情况做不同的事，`match` 就是最自然的工具。

**`format!` 和着色链。** `format!("!! urgent".red().bold())` 先用 `colored` 的 `.red().bold()` 方法给字符串加上 ANSI 样式，再用 `format!` 把它转成普通 `String`（因为 `println!` 的格式参数不能直接接受 colored 的返回类型）。`.dimmed()` 降低 ID 的视觉权重，`.strikethrough()` 给已完成内容加删除线，`.green()` 让 `✓ done` 变成绿色。比 `\x1b[32m` 可读十倍。

> [!ASIDE]
> **什么是 trait？** 如果你来自面向对象语言，trait 类似于接口（interface）。它定义一组方法签名，任何类型都可以 `impl` 它。`Colorize` trait 定义了 `.green()`、`.red()` 等方法，`colored` crate 为 `&str` 和 `String` 提供了实现。你用 `use colored::Colorize;` 把 trait 引入作用域后，这些方法就能在任何字符串上调用。Rust 标准库里的 `Iterator`、`Display`、`Clone` 都是 trait——你在 part 1 中用过的 `#[derive(Clone)]` 就是让编译器自动为你的类型实现 `Clone` trait。

## 给待办加上优先级

现在 `print_todos` 引用了 `t.priority`，但 `Todo` 结构体还没有这个字段。来加上它——这也是 part 1 练习 1 的正式实现。

修改结构体，添加 `priority` 字段：

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Todo {
    id: u32,
    content: String,
    completed: bool,
    created_at: String,
    #[serde(default)]
    priority: u8,
}
```

`#[serde(default)]` 是一个 serde 属性注解——它告诉反序列化器：如果 JSON 中没有 `priority` 字段，使用 `u8` 的默认值 `0`。这个注解解决了一个真实问题：你现有的 `todos.json` 里的条目没有 `priority`，如果没有 `#[serde(default)]`，反序列化会因为缺少字段而失败——就像 part 1 练习 3 中 `String` 改 `DateTime` 时遇到的那个问题一样。

> [!HEADS-UP]
> 这就是数据迁移的实际代价。每次给结构体添加新字段，旧 JSON 文件不包含它，反序列化就会报错。`#[serde(default)]` 是最简单的应对策略——给新字段一个合理的默认值。如果你需要更复杂的迁移逻辑（比如从旧格式转换），可以用 `#[serde(deserialize_with = "...")]` 指定自定义反序列化函数，但那是后续话题。

更新 `Todo::new()` 接受优先级参数：

```rust
impl Todo {
    fn new(id: u32, content: String, priority: u8) -> Self {
        let now = chrono::Local::now().format("%Y-%m-%d %H:%M").to_string();
        Todo {
            id,
            content,
            completed: false,
            created_at: now,
            priority,
        }
    }
}
```

注意构造函数签名从 `fn new(id: u32, content: String)` 变成了 `fn new(id: u32, content: String, priority: u8)`。编译器会在每个调用 `Todo::new()` 的地方报错——这正是 part 1 结尾反思题中提到的穷举检查的威力：你改了函数签名，编译器自动找到所有需要更新的地方。

在 `Commands` 枚举中，给 `Add` 变体添加优先级选项：

```rust
#[derive(Subcommand)]
enum Commands {
    /// Add a new todo
    Add {
        /// The task description
        #[arg(required = true)]
        content: String,
        /// Priority: 0 = normal, 1 = high, 2 = urgent
        #[arg(short, long, default_value_t = 0)]
        priority: u8,
    },
    // ... List, Done, Remove 保持不变
}
```

`#[arg(short, long, default_value_t = 0)]` 生成 `-p` 和 `--priority` 选项，默认值为 `0`。`default_value_t` 中的 `t` 表示这个值是一个 Rust 表达式（这里是字面量 `0`），而不是字符串。clap 会自动把字符串输入解析为 `u8`——用户输入 `-p 2` 时，你拿到的是整数 `2`，不是字符串 `"2"`。

> [!ASIDE]
> **`default_value_t` 和 `default_value` 的区别。** `default_value = "0"` 接受字符串，clap 在运行时把字符串解析为目标类型。`default_value_t = 0` 直接接受一个类型匹配的值。对于 `u8`、`bool` 这种简单类型，`default_value_t` 更直接；对于 `PathBuf` 或自定义类型，`default_value` 更灵活，因为它走的是字符串解析路径。

现在更新 `cmd_add` 来传递优先级，并用 `colored` 替换裸 ANSI 码：

```rust
fn cmd_add(todos: &mut Vec<Todo>, content: String, priority: u8) {
    let new_id = todos.iter().map(|t| t.id).max().unwrap_or(0) + 1;
    let todo = Todo::new(new_id, content, priority);
    let priority_info = if priority > 0 {
        format!("  [{}]", match priority {
            2 => "urgent".red().bold().to_string(),
            1 => "high".yellow().to_string(),
            _ => "normal".to_string(),
        })
    } else {
        String::new()
    };
    println!("{} Added #{}: {}{}", "✓".green(), todo.id, todo.content, priority_info);
    todos.push(todo);
}
```

这行代码值得仔细看：`todos.iter().map(|t| t.id).max().unwrap_or(0) + 1`。这是 Rust 迭代器链的经典模式，你在 part 1 中见过，现在再走一遍：

1. `todos.iter()` — 把 `Vec<Todo>` 变成 `Iterator<Item = &Todo>`，一个遍历每个 `Todo` 引用的迭代器
2. `.map(|t| t.id)` — 闭包 `|t| t.id` 从每个 `&Todo` 中提取 `id` 字段，迭代器变成 `Iterator<Item = u32>`
3. `.max()` — 消耗迭代器，返回 `Option<u32>`：有元素就是 `Some(最大id)`，空迭代器就是 `None`
4. `.unwrap_or(0)` — 如果是 `None`（列表为空），返回 `0`
5. `+ 1` — 新 ID 比现有最大 ID 大 1

这个链条在 Rust 中**极其常见**。过滤、映射、聚合——任何需要遍历集合并产出结果的场景都是这个模式。你会在后续每个部分反复遇到它。

最后，重写 `cmd_list` 使用 `print_todos`：

```rust
fn cmd_list(todos: &[Todo], all: bool) {
    let items: Vec<&Todo> = if all {
        todos.iter().collect()
    } else {
        todos.iter().filter(|t| !t.completed).collect()
    };
    let label = if all { "All" } else { "Pending" };
    print_todos(label, &items);
}
```

和 part 1 的 `cmd_list` 对比：过滤逻辑完全相同——`todos.iter().filter(|t| !t.completed).collect()` 这个迭代器链是你在 Rust 中做"从集合中筛选子集"的标准姿势。不同的是打印逻辑被委托给了 `print_todos`。`&items` 把 `Vec<&Todo>` 自动转换为 `&[&Todo]`——这是 part 1 中讲过的 deref coercion，Rust 自动把 `&Vec<T>` 变成 `&[T]`。

## 搜索子命令

现在添加 `search` 命令——这是 part 1 练习 2 的正式实现。

在 `Commands` 枚举中添加新变体：

```rust
#[derive(Subcommand)]
enum Commands {
    // ... Add, List, Done, Remove 保持不变
    /// Search todos by keyword
    Search {
        /// The keyword to search for
        keyword: String,
    },
}
```

`///` 注释和 `keyword: String` 的写法你应该已经很熟悉了——和 `Add` 的 `content` 一样，它是一个位置参数。

添加 `cmd_search` 函数，放在 `cmd_remove` 之后：

```rust
fn cmd_search(todos: &[Todo], keyword: &str) {
    let matches: Vec<&Todo> = todos.iter()
        .filter(|t| t.content.contains(keyword))
        .collect();
    let label = format!("Search: \"{}\"", keyword);
    print_todos(&label, &matches);
}
```

四行代码，但每一行都浓缩了 Rust 的核心模式。让我把迭代器链拆开：

```
todos.iter()                    // Iterator<Item = &Todo>
    .filter(|t| t.content.contains(keyword))  // 只保留 content 包含关键词的
    .collect()                  // 把迭代器收集为 Vec<&Todo>
```

**`.filter()`** 接受一个闭包，闭包返回 `bool`。返回 `true` 的元素保留，返回 `false` 的被丢弃。`|t| t.content.contains(keyword)` 中，`t` 是 `&&Todo`（因为 `iter()` 产生 `&Todo`，`filter` 再包一层引用），Rust 的自动解引用让你直接写 `t.content` 而不需要 `(**t).content`。

**`.collect()`** 是一个"终点操作"——它消耗迭代器并把结果收集到目标容器中。目标类型由变量声明决定：`let matches: Vec<&Todo>` 告诉编译器收集为 `Vec<&Todo>`。如果你改成 `let matches: Vec<Todo>`（不带引用），编译器会报错，因为 `.filter()` 产生的是 `&&Todo`，不能直接变成 `Todo`——除非你加一步 `.cloned()`。

> [!ASIDE]
> **为什么 `filter` 的闭包参数是 `&&Todo`？** 因为 `iter()` 产生 `&Todo`，而 `filter` 传给闭包的是对迭代器元素的引用，所以多了一层 `&`。Rust 的自动解引用（auto-deref）在大多数场景下帮你处理了这个问题——`t.content` 实际上是 `(*(*t)).content` 的简写，但你不需要手动解。如果你觉得这层引用让人困惑，记住一个实用规则：在迭代器链的闭包中，闭包参数总是比迭代器的 `Item` 类型多一个 `&`。

最后，在 `main` 的 `match` 中添加 `Search` 分支。编译器会在你保存文件时提醒你——这就是 part 1 结尾反思题中那个问题的答案：`match` 的穷举检查确保你不可能漏掉新命令。

```rust
fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();
    let mut todos = load_todos()?;

    match cli.command {
        Commands::Add { content, priority } => {
            cmd_add(&mut todos, content, priority);
            save_todos(&todos)?;
        }
        Commands::List { all } => {
            cmd_list(&todos, all);
        }
        Commands::Done { id } => {
            cmd_done(&mut todos, id);
            save_todos(&todos)?;
        }
        Commands::Remove { id } => {
            cmd_remove(&mut todos, id);
            save_todos(&todos)?;
        }
        Commands::Search { keyword } => {
            cmd_search(&todos, &keyword);
        }
    }

    Ok(())
}
```

注意两个变化：`Commands::Add` 现在解构出 `content` 和 `priority` 两个字段；`main` 的签名变成了 `-> anyhow::Result<()>`，`save_todos` 和 `load_todos` 后面出现了 `?`。这是下一节的内容。

## 用 anyhow 替换 expect

到目前为止，`load_todos` 和 `save_todos` 用 `.expect()` 处理所有错误——文件读不了就 panic，JSON 损坏就 panic。对个人工具来说这不是大问题，但 part 1 练习 3 揭示了一个真实的痛点：当你把 `created_at` 从 `String` 改成 `DateTime<Local>`，旧的 JSON 格式不兼容，`load_todos` **无声地返回空列表**而不是告诉你发生了什么。

[`anyhow`](https://docs.rs/anyhow/latest/anyhow/) 解决这个问题。它提供一个统一的 `anyhow::Result<T>` 类型（等价于 `Result<T, anyhow::Error>`），一个 `?` 操作符来自动传播错误，和一个 `.context()` 方法来给错误添加人类可读的说明。

修改 `load_todos` 的签名和实现：

```rust
use anyhow::{Context, Result};

fn load_todos() -> Result<Vec<Todo>> {
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
```

> [!HEADS-UP]
> `Ok` 的 `k` 是**小写**的。Rust 的 `Result` 枚举有两个变体：`Ok(T)` 和 `Err(E)`——只有首字母大写。写成 `OK(todos)`（大写 K）会报 `cannot find function 'OK' in this scope`。这是一个非常常见的笔误，几乎每个 Rust 开发者都踩过。记住：`Ok`，不是 `OK`。

三个关键变化。

**返回类型从 `Vec<Todo>` 变为 `Result<Vec<Todo>>`。** `Result<T>` 是 anyhow 的类型别名，展开后是 `std::result::Result<T, anyhow::Error>`。函数现在明确告诉调用者："我可能会失败"。

**`.context("Failed to read todos file")?` 替换 `.expect("...")`。** 区别在于行为：`.expect()` 在 `Err` 时直接 panic（程序崩溃）；`.context()` 给错误添加一层说明文字，然后 `?` 把 `Err` 传播给调用者。调用者决定怎么处理这个错误——打印、重试、或忽略。

**`unwrap_or_default()` 保留。** JSON 解析失败时仍然返回空列表——这是一个刻意的设计选择。如果你想在 JSON 损坏时也报错而不是静默回退，把 `unwrap_or_default()` 换成 `.context("Failed to parse todos JSON")?` 就行。

> [!DESIGN-NOTE]
> **`?` 操作符的工作原理。** `?` 是一个语法糖：`expr?` 等价于 `match expr { Ok(v) => v, Err(e) => return Err(e.into()) }`。当 `expr` 是 `Ok` 时，`?` 提取内部的值；当是 `Err` 时，`?` 把错误**立即返回**给调用者——当前函数的剩余代码不执行。`.into()` 调用让不同类型的错误可以自动转换为 `anyhow::Error`：`io::Error`、`serde_json::Error` 等都实现了 `Into<anyhow::Error>`。这意味着你在一个函数里可以混用不同来源的 `?`，anyhow 统一处理。

修改 `save_todos` 同理：

```rust
fn save_todos(todos: &[Todo]) -> Result<()> {
    let path = data_file();
    let json = serde_json::to_string_pretty(todos)
        .context("Failed to serialize todos")?;
    fs::write(&path, json)
        .context("Failed to write todos file")?;
    Ok(())
}
```

注意 `Ok(())` 的写法——`()` 是 Rust 的"单元类型"（unit type），类似于其他语言的 `void`。`Ok(())` 表示"操作成功，没有返回值"。

现在更新 `main` 函数。你已经看到了上面的完整代码——签名变为 `fn main() -> anyhow::Result<()>`，所有 `save_todos` 和 `load_todos` 调用后面加了 `?`，函数末尾加 `Ok(())`。

当错误发生时，anyhow 会打印一条清晰的错误链：

```
Error: Failed to read todos file

Caused by:
    Permission denied (os error 13)
```

这比 `thread 'main' panicked at 'Failed to read todos file'` 有用得多——用户知道**什么操作失败了**以及**底层原因是什么**。

> [!ASIDE]
> **`anyhow` vs `thiserror`。** anyhow 是应用级错误处理——用一个统一的 `Error` 类型装所有错误。[`thiserror`](https://docs.rs/thiserror/latest/thiserror/) 是库级错误处理——让你定义精确的错误枚举，每个变体有自己的类型。如果你在写一个给别人用的 crate，用 `thiserror` 让调用者能精确匹配每种错误。如果你在写应用（像 `tasky` 这样的终端工具），anyhow 更简单，也是更常见的选择。两者可以共存：库用 `thiserror` 定义错误，应用用 `anyhow` 消费它们。

## 检查点

> [!PREDICT]
> 在运行之前想一想：如果你之前做过 part 1 的练习 3（`created_at` 改为 `DateTime<Local>`）并且 `todos.json` 里有旧格式的日期字符串，现在运行 `tasky list` 会怎样？`unwrap_or_default()` 在这里做了什么？

**运行以下命令验证你目前的工作：**

```bash
cargo build
```

预期输出：编译成功，无错误（可能有 `unused` 警告，忽略即可）。

```bash
cargo run -- add -p 2 "修复生产环境 bug"
```

预期输出：

```
✓ Added #1: 修复生产环境 bug  [urgent]
```

```bash
cargo run -- add "学习 Rust 迭代器"
```

预期输出：

```
✓ Added #2: 学习 Rust 迭代器
```

```bash
cargo run -- list
```

预期输出（优先级 2 的条目有红色 urgent 标记）：

```
  Pending (2):
    [1] 修复生产环境 bug  (2026-06-14 09:15)  !! urgent
    [2] 学习 Rust 迭代器  (2026-06-14 09:16)
```

```bash
cargo run -- search "Rust"
```

预期输出：

```
  Search: "Rust" (1):
    [2] 学习 Rust 迭代器  (2026-06-14 09:16)
```

```bash
cargo run -- done 1
```

预期输出：

```
✓ Done #1: 修复生产环境 bug
```

```bash
cargo run -- list --all
```

预期输出（已完成的条目有删除线和绿色 ✓）：

```
  All (2):
    [1] 修复生产环境 bug  ✓ done  (2026-06-14 09:15)  !! urgent
    [2] 学习 Rust 迭代器  (2026-06-14 09:16)
```

**可能的错误：**

- 如果看到 `no field 'priority' on type 'Todo'`，你可能忘了在 `Todo` 结构体中添加 `priority` 字段——确保 `#[serde(default)] priority: u8` 在结构体定义中。
- 如果看到 `this function takes X arguments but Y arguments were supplied`（关于 `Todo::new`），你可能没有更新 `cmd_add` 中的调用——`Todo::new(new_id, content, priority)` 需要三个参数。
- 如果看到 `the trait bound 'DateTime<Local>: Deserialize<'_>' is not satisfied`，你可能做过练习 3 但没有给 chrono 加 `features = ["serde"]`——检查 `Cargo.toml`。

## 接下来

你的 `tasky` 现在有了优先级、搜索和优雅的错误处理。但它还是一个单文件程序——所有代码挤在 `main.rs` 里。随着功能增加，这个文件会越来越长。后续部分将回答：如何把代码拆分到多个模块文件（`todo.rs`、`cli.rs`、`storage.rs`）？如何编写集成测试验证 CLI 的端到端行为？如何用 `cargo install` 发布让任何人都能一条命令安装你的工具？

在继续之前：本部分引入了三种使用迭代器的场景——`cmd_add` 中用 `.iter().map().max()` 计算新 ID、`cmd_list` 中用 `.iter().filter().collect()` 筛选待办、`cmd_search` 中用同样的链条搜索关键词。用你自己的话描述这个三步模式的共同结构：每一步（`iter`、`map`/`filter`、`collect`/`max`）各自做了什么？为什么 Rust 不直接提供一个 `.find_all(condition)` 方法，而是要拆成 `.filter().collect()` 两步？

## 练习

1. **给 `list` 添加 `--priority` 过滤。** 添加 `#[arg(short, long)] priority: Option<u8>` 到 `Commands::List`。`Option<u8>` 意味着这个选项是可选的——用户不传就是 `None`（显示所有优先级），传了就是 `Some(n)`（只显示优先级 ≥ n 的条目）。在 `cmd_list` 的 `filter` 链中加一个条件：`t.priority >= p`。

2. **实现 `edit` 子命令。** 接受 `id: u32` 和 `content: String`，更新指定 ID 的待办内容。提示：用 `iter_mut().find(|t| t.id == id)` 获取可变引用，然后修改 `.content` 字段。这个模式和 `cmd_done` 完全相同——只是修改的字段不同。

3. **把 `load_todos` 中的 `unwrap_or_default()` 替换为 `.context()` 调用。** 这样 JSON 损坏时不再静默返回空列表，而是打印一条明确的错误信息。运行测试：手动把 `todos.json` 的内容改成无效 JSON，看看 anyhow 的错误输出是什么样的。

4. **给 `print_todos` 的已完成条目添加完成时间。** 在 `Todo` 结构体中添加 `#[serde(default)] completed_at: Option<String>` 字段，在 `cmd_done` 中记录完成时间（用 `chrono::Local::now().format(...).to_string()`），在 `print_todos` 中如果 `completed_at` 是 `Some(time)` 就显示在 `✓ done` 后面。

5. **把 `main.rs` 拆分为两个文件。** 创建 `src/todo.rs`，把 `Todo` 结构体、`impl Todo`、`load_todos`、`save_todos`、`data_file` 移过去，在顶部加 `pub` 可见性标记。在 `main.rs` 中用 `mod todo; use todo::{Todo, load_todos, save_todos};` 导入。这是 Rust 模块系统的第一步——你会在后续部分深入这个话题。

## 来源

1. [anyhow documentation](https://docs.rs/anyhow/latest/anyhow/) —— 应用级错误处理框架，本部分使用 `Result`、`?`、`.context()` 替换 `.expect()`。
2. [thiserror documentation](https://docs.rs/thiserror/latest/thiserror/) —— 库级错误类型定义框架，与 anyhow 互补。
3. [colored crate documentation](https://docs.rs/colored/latest/colored/) —— 终端字符串着色库，`Colorize` trait 提供链式着色 API。
4. [serde field attributes](https://serde.rs/field-attrs.html) —— `#[serde(default)]` 等字段级注解的完整参考，用于处理缺失字段的数据迁移问题。
5. [Rust iterators](https://doc.rust-lang.org/std/iter/trait.Iterator.html) —— 标准库迭代器 trait 文档，`map`、`filter`、`collect`、`max` 等方法的权威参考。
