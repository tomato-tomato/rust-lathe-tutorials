# 给 grep-search 装上一个开关：反选、选项解析与一次诚实的重构

一把只会干一件事的工具，用起来是顺手的。到了要它干第二件事的时候，问题才会浮出来——不是功能不够，是骨架撑不住。

上一部分结束时，你手里的 `grep-search` 已经能搜一个文件、报像样的错误、给脚本正确的退出码，还有三个测试守着契约。练习你也动了手：大小写开关、行号、错误分类。这些改动里，有做对的，有做得可以但留下了债的，还有一道没做完。

这一部分先把账算清楚——**验收你已经写下的代码**，然后补上最后那道练习：反向匹配 `-v`。而补它的过程中，我们会撞上一个比 `-v` 本身更重要的问题：参数从两个变成三个、四个的时候，`main` 里那套"数个数、按下标取"的写法还能不能站住。

## 你将构建什么

一个能反选的 `grep-search`：`cargo run -- -v berry fruits.txt` 输出 `fruits.txt` 里**不含** `berry` 的每一行，退出码语义照旧（有选中行 0、无选中行 1、出错 2）。参数解析被收进一个 `parse_args` 函数，产出一个有名字的 `Options` 结构；筛选逻辑回到它该管的事——判断哪一行被选中，并由测试覆盖正选、反选、大小写及其组合。

## 前置条件

- Rust 工具链 1.85 或更新。本教程按 **Rust 1.98** 写作。
- 完成了第 1 部分，并动手做过它的练习 1–3。下面是验收，你需要手里有那份代码。
- `fruits.txt` 仍在项目目录（`apple`、`apricot`、`banana`、`blueberry`、`cherry`）。

> [!RECALL]
> 继续之前，凭记忆回答两个问题，别翻上一部分：(1) `grep-search` 的三个退出码各代表什么？(2) `matching_lines` 为什么被写成"纯函数"，测试为什么能直接调它而不需要文件和终端？这两个答案本部分都要用到——反选会改变"选中"的定义，而我们会改动那个纯函数的签名。

## 先验收：练习做对了多少

我从你的仓库里读到了 `grep-search/src/main.rs`，并且**用你机器上的工具链实测过**（`rustc 1.98.1` / `cargo 1.98.1`）。下面每条结论都注明了它来自实测还是推演。

**练习 3（区分错误种类）——做对了。** 这段是标准答案的水平：

```rust
match err.kind() {
    ErrorKind::NotFound => {
        eprintln!("grep-search: {path}: 没有这个文件");
    }
    _ => eprintln!("grep-search: {path}: {err}"),
}
```

`ErrorKind::NotFound` 单独招呼、其余走通配分支，而且 `return ExitCode::from(2)` 的位置没动——错误路径的退出码没有被改坏。这正好印证了第 1 部分那个设计点：错误分不分种类是"消息"的事，不是"退出码"的事。

**练习 1（大小写不敏感）——意图对，写法脆。** 你写的是：

```rust
let cur_pattern = if ignore_case {
    &pattern.to_lowercase()
} else {
    pattern
};
```

`pattern.to_lowercase()` 造出一个临时的 `String`，`&` 只是借它。**实测结果：这段代码在 1.98 下编译通过。** 但它恰好踩在两个现代编译器行为上，值得你知道它们是什么：

- **它绕过了历史上有名的坑。** 在 `if` 分支里借用临时值，老版本编译器会报 [`E0716: temporary value dropped while borrowed`](https://doc.rust-lang.org/error_codes/E0716.html)——临时值在语句结束时就被回收，借出去的东西却还要用。近年的 Rust 把临时生命周期延长规则扩展到了 `if` 块（[PR #146098](https://github.com/rust-lang/rust/pull/146098)），这段代码才站得住。换句话说：它已经"从一定报错"变成了"依赖编译器版本"。
- **两个分支的类型其实不一样。** 一个分支是 `&String`，另一个是 `&str`。这里能过，靠的是编译器的分支统一逻辑顺手把 `&String` 解引用转成了 `&str`——省了你的事，也意味着你读这段代码时得多想一层"这俩类型并不相同"。

"能跑"和"经得起改"是两回事：通过的条件握在编译器的实现细节里，而不是握在你手里。改法很简单，而且让类型本来就对齐、值本来就自有。这一部分会带你把它改掉。

**练习 2（打印行号）——行为实现了，但把测试弄红了。** 你的实现把行号直接拼进了筛选函数的返回值：

```rust
hits.push(format!("{}: {}", number + 1, line.to_string()));
```

输出确实变成了 `3: banana`，符合练习的字面要求。但你的测试没有跟着改——测试断言的是 `vec!["banana"]`，而函数现在返回 `"3: banana"`。**实测确认（`cargo test`）：1 passed，3 failed**，报告就是断言与实现的正面冲突：

```
left: ["3: banana"]   right: ["banana"]     ← keeps_only_lines_containing_the_pattern
left: ["2: apple"]    right: ["apple"]      ← matching_is_case_sensitive
left: ["1: Apple"]    right: ["Apple"]      ← matching_is_case_not_sensitive
```

`returns_nothins_when_no_line_matches` 因为断言的是"空"，仍然通过（顺带一提，那个测试名里的 `nothins` 是笔误）。

> [!HEADS-UP]
> 先跑一次 `cargo test` 确认你看到的和我读到的一致——**红了别慌，这是好消息**。它说明测试真的在替你守着契约：改了行为，测试立刻报警，而不是等用户在脚本里踩到。本部分结束前我们会让它们全绿。

**练习 4（`-v` 反向匹配）——没做。** 这正是本部分的正题。

## 练习 4 的答案长什么样：一个开关，两个方向

先看需求，别急着写代码。真实 grep 的 `-v`（`--invert-match`）做的事只有一件：**它反转"哪些行被选中"**。默认选中的是含模式的行；加了 `-v`，选中的变成不含模式的行——输出的、参与退出码判断的，都是这个"选中集合"。

这里有个术语值得记住，因为 grep 的手册就是这么措辞的：`-v` 翻转的是**选中行**（selected line）的集合，而退出码规则本身纹丝不动——[GNU grep 手册](https://www.gnu.org/software/grep/manual/html_node/Matching-Control.html)里 `-v, --invert-match` 的定义是"反转匹配的意义"；[Exit Status](https://www.gnu.org/software/grep/manual/html_node/Exit-Status.html) 一节说的仍是"有行被选中则 0"。所以 `grep -v mango fruits.txt` 里每一行都不含 `mango`，全部被选中——退出码 0。**契约没变，变的是"选中"的含义。**

> [!ASIDE]
> "selected" 这个词不是修辞。grep 的着色配置文档里干脆就用它来定义 `-v`：默认情况下 **selected lines 就是 matching lines**，加了 `-v` 就变成 **non-matching lines**。一个词把两种模式统一了，这也是我们下面给函数改名的理由。

最直接的实现——也是大多数人第一反应写出的样子——是在 `main` 里就地加分支：

```rust
if args[1] == "-v" {
    if args.len() != 4 { /* 报用法 */ }
    let hits = non_matching_lines(&text, args[2].as_str());
    // ...
} else {
    if args.len() != 3 { /* 报用法 */ }
    let hits = matching_lines(&text, args[1].as_str());
    // ...
}
```

它现在能跑。但数一数：参数个数判断写了两遍、下标写在两处、每加一个开关就多一层分支。等你想同时支持 `-v` 和 `-i`，分支就不是两层而是四层，再往下是八个。**问题不在功能，在形状**——同一件事（把 argv 翻译成"我该怎么做"）被摊在了本该只管流程的 `main` 里。

## 把命令行翻译成一个结构：parse_args

正确的形状是：让一个专门的函数吃掉原始参数，吐出一个**有名字的配置**。先定义这个配置长什么样：

```rust
struct Options {
    pattern: String,
    path: String,
    invert: bool,
    ignore_case: bool,
}
```

四个字段，都是一眼能读懂的意图——没有 `args[1]` 这种"第几个参数"的位置记忆。注意这些字段**自己拥有数据**（`String` 而不是 `&str`），所以 `Options` 不绑定任何外部的生命周期，传到哪里都成立。这一点直接呼应上面练习 1 的毛病：函数往外交东西时，让它拥有值，就少一类"借用临时值"的麻烦。

然后是翻译函数。它只做一件事：扫一遍参数，把开关记下来，把剩下的两个位置参数当模式与文件：

```rust
fn parse_args(args: &[String]) -> Result<Options, String> {
    let mut invert = false;
    let mut ignore_case = false;
    let mut rest: Vec<&str> = Vec::new();

    for arg in args.iter().skip(1) {
        match arg.as_str() {
            "-v" => invert = true,
            "-i" => ignore_case = true,
            _ => rest.push(arg.as_str()),
        }
    }

    if rest.len() != 2 {
        return Err("用法：grep-search [-v] [-i] <字面模式> <文件>".to_string());
    }

    Ok(Options {
        pattern: rest[0].to_string(),
        path: rest[1].to_string(),
        invert,
        ignore_case,
    })
}
```

几处值得停一下：

- 返回类型是 `Result<Options, String>`——失败时给一句人话。这是你在第 1 部分学过两次的 `Result`，第三次见面了：读文件用它，现在解析参数也用它。**凡是可能失败的事，返回值里都带着失败的可能**，这条不因场景改变。
- `args.iter().skip(1)` 跳过程序名（下标 0 那个老规矩），从真正的参数开始扫。开关 `-v`/`-i` 被吃掉，其余按顺序进 `rest`。
- 数量检查从"`args.len() != 3`"升级成"`rest.len() != 2`"——**位置参数还剩几个**，与开关出现的位置、个数都无关。这就是形状带来的好处：`grep-search -v pattern file` 与 `grep-search pattern file -v` 都能工作。

`main` 开头随之变得很薄：

```rust
fn main() -> ExitCode {
    let args: Vec<String> = env::args().collect();

    let opts = match parse_args(&args) {
        Ok(opts) => opts,
        Err(message) => {
            eprintln!("{message}");
            return ExitCode::from(2);
        }
    };
    // ...下面接着读文件
}
```

错误路径与第 1 部分打开文件时**一模一样**：`eprintln!` 写 stderr、`return ExitCode::from(2)`。你不是在学新东西，是在把学过的东西摆到新位置。

> [!DESIGN-NOTE]
> **为什么不直接在 `main` 里判断 `-v`，非要抽一个 `parse_args`？**
> 因为"读取意图"和"执行意图"是两件事，混在一起的代价随选项数量超线性增长。抽出来之后有三重收益：`main` 只讲流程（解析 → 读文件 → 筛选 → 输出 → 定退出码）；选项的增删只改 `parse_args` 一个地方；最重要的——`Options` 让下游函数不再依赖"参数在第几个位置"，你给 `selected_lines` 传的是一个语义对象。
> 顺带说，这也解释了你上次学完的 `todo-cli-rust` 里那个 `clap` 在做什么。那门课里你写的是 `#[derive(Parser)] struct Cli { ... }` 加一个 `Commands` 枚举，`tasky add "买牛奶" --priority 1` 就自动变成一个填好字段的结构体。`clap` 干的事和我们这个 `parse_args` 是同一件：**把 argv 翻译成一个结构**。区别只是它替你处理了 `-h`、`--long` 选项、子命令、类型转换这些边角。先手写过一遍，再回头看 `#[derive(Parser)]`，你就不是在"照着文档抄"，而是在看一个你已经理解了的机器。

## 让匹配函数回到本分：selected_lines

现在处理那道旧债。你的筛选函数现在既判断命中、又拼行号字符串，还负责大小写。它做三件事，于是测试一改就红。把它拆清楚：**筛选函数回答"哪一行被选中、它是第几行"，格式化留给打印的人。**

顺便改名。`matching_lines` 这个名字在只有正向匹配时没问题，但一旦有了 `-v`，"匹配"和"选中"就是两个概念了——grep 手册用的词是 selected。名字跟着语义走：

```rust
fn selected_lines(text: &str, opts: &Options) -> Vec<(usize, String)> {
    let needle = if opts.ignore_case {
        opts.pattern.to_lowercase()
    } else {
        opts.pattern.clone()
    };

    let mut hits = Vec::new();

    for (index, line) in text.lines().enumerate() {
        let haystack = if opts.ignore_case {
            line.to_lowercase()
        } else {
            line.to_string()
        };

        let matched = haystack.contains(&needle);
        let selected = if opts.invert { !matched } else { matched };

        if selected {
            hits.push((index + 1, line.to_string()));
        }
    }

    hits
}
```

逐点看，每一个改动都在还上面欠下的账：

- **签名从 `(text, pattern, ignore_case)` 变成 `(text, opts)`。** 参数不再随选项增加而变长——`Options` 已经是那个"有名字的集合"。这也顺带把练习 1 的第三个参数问题一并收编了。
- **练习 1 的两处毛病在这里被消掉。** 看 `haystack`：两个分支分别是 `line.to_lowercase()`（`String`）和 `line.to_string()`（`String`），**类型相同、各自拥有值**——不再有 `&String` 对 `&str` 的类型统一问题，也不存在借用临时值。这就是我开头说的改法：不去调和借用，直接让两边都拥有值。
- **`needle` 只算一次。** 模式是固定的，不该每行都做一次小写转换。这个念头第 1 部分就有过——那时我们讨论过 `to_string()` 每行复制一次的浪费。区别是这里的浪费更容易避免：把不变的东西挪到循环外。
- **`matched` 与 `selected` 分开写。** `matched` 是"这一行含不含模式"，`selected` 是"这一行要不要"，`-v` 只影响后者。写成一行的 `matched != opts.invert` 也正确，但两个名字摆在眼前，你读代码时不用在脑子里做异或。
- **返回 `Vec<(usize, String)>`——行号跟着选中的行一起出来。** 行号必须在扫描时记录（否则事后无从知道它来自第几行），所以它属于筛选的产物；但**拼成 `"3: banana"` 这种字符串是输出格式**，属于 `main`。这就是"把格式化留给打印的人"的意思。

`main` 里接收它，用模式解构把元组拆开：

```rust
    let hits = selected_lines(&text, &opts);

    for (number, line) in &hits {
        println!("{number}: {line}");
    }
```

`(number, line)` 是元组模式——`for` 每轮把 `(usize, String)` 拆成两个名字，和 Python 的 `a, b = pair` 是同一件事。

> [!NOTE]
> `line.to_lowercase()` 每行都会分配一个新 `String`。对 `fruits.txt` 无所谓，对几十万行就是实打实的开销。更省的办法是用 `Cow<str>`（借用与拥有二选一），或只在发现模式含大写时才做转换——这是进阶话题，等你处理大文件时再回来。今天先记住"知道它花钱、也知道钱花在哪"。

到这里，完整的 `src/main.rs` 是：

```rust
use std::env;
use std::fs;
use std::io::ErrorKind;
use std::process::ExitCode;

struct Options {
    pattern: String,
    path: String,
    invert: bool,
    ignore_case: bool,
}

fn main() -> ExitCode {
    let args: Vec<String> = env::args().collect();

    let opts = match parse_args(&args) {
        Ok(opts) => opts,
        Err(message) => {
            eprintln!("{message}");
            return ExitCode::from(2);
        }
    };

    let text = match fs::read_to_string(&opts.path) {
        Ok(text) => text,
        Err(err) => {
            match err.kind() {
                ErrorKind::NotFound => eprintln!("grep-search: {}: 没有这个文件", opts.path),
                _ => eprintln!("grep-search: {}: {err}", opts.path),
            }
            return ExitCode::from(2);
        }
    };

    let hits = selected_lines(&text, &opts);

    for (number, line) in &hits {
        println!("{number}: {line}");
    }

    if hits.is_empty() {
        ExitCode::from(1)
    } else {
        ExitCode::SUCCESS
    }
}

fn parse_args(args: &[String]) -> Result<Options, String> {
    let mut invert = false;
    let mut ignore_case = false;
    let mut rest: Vec<&str> = Vec::new();

    for arg in args.iter().skip(1) {
        match arg.as_str() {
            "-v" => invert = true,
            "-i" => ignore_case = true,
            _ => rest.push(arg.as_str()),
        }
    }

    if rest.len() != 2 {
        return Err("用法：grep-search [-v] [-i] <字面模式> <文件>".to_string());
    }

    Ok(Options {
        pattern: rest[0].to_string(),
        path: rest[1].to_string(),
        invert,
        ignore_case,
    })
}

fn selected_lines(text: &str, opts: &Options) -> Vec<(usize, String)> {
    let needle = if opts.ignore_case {
        opts.pattern.to_lowercase()
    } else {
        opts.pattern.clone()
    };

    let mut hits = Vec::new();

    for (index, line) in text.lines().enumerate() {
        let haystack = if opts.ignore_case {
            line.to_lowercase()
        } else {
            line.to_string()
        };

        let matched = haystack.contains(&needle);
        let selected = if opts.invert { !matched } else { matched };

        if selected {
            hits.push((index + 1, line.to_string()));
        }
    }

    hits
}
```

注意 `if hits.is_empty()` 还是 `main` 的最后一个表达式，`ExitCode` 那三个刻度一个没动——重构换的是形状，不是契约。这一点值得说透：**好的重构让行为不变**。如果改完发现退出码变了，那不是重构，是改需求。

## 测试跟上：从三个到六个

签名改了，旧的测试会**编译不过**（`matching_lines` 这个名字都不存在了）。这是编译器在帮你——它把每一个需要跟着改的调用点都指给了你。

测试里反复构造 `Options` 很啰嗦，加一个小助手：

```rust
#[cfg(test)]
mod tests {
    use super::{selected_lines, Options};

    fn opts(pattern: &str, invert: bool, ignore_case: bool) -> Options {
        Options {
            pattern: pattern.to_string(),
            path: "fruits.txt".to_string(),
            invert,
            ignore_case,
        }
    }

    #[test]
    fn keeps_only_lines_containing_the_pattern() {
        let text = "apple\napricot\nbanana";
        assert_eq!(
            selected_lines(text, &opts("an", false, false)),
            vec![(3, "banana".to_string())]
        );
    }

    #[test]
    fn returns_nothing_when_no_line_matches() {
        let text = "apple\nbanana";
        assert!(selected_lines(text, &opts("mango", false, false)).is_empty());
    }

    #[test]
    fn matching_is_case_sensitive() {
        let text = "Apple\napple";
        assert_eq!(
            selected_lines(text, &opts("apple", false, false)),
            vec![(2, "apple".to_string())]
        );
    }

    #[test]
    fn can_ignore_case() {
        let text = "Apple\napricot";
        assert_eq!(
            selected_lines(text, &opts("apple", false, true)),
            vec![(1, "Apple".to_string())]
        );
    }

    #[test]
    fn invert_keeps_lines_without_the_pattern() {
        let text = "apple\napricot\nbanana";
        assert_eq!(
            selected_lines(text, &opts("an", true, false)),
            vec![(1, "apple".to_string()), (2, "apricot".to_string())]
        );
    }

    #[test]
    fn invert_and_ignore_case_together() {
        let text = "Apple\napricot";
        assert_eq!(
            selected_lines(text, &opts("APPLE", true, true)),
            vec![(2, "apricot".to_string())]
        );
    }
}
```

六个用例，每个钉住一条你已经实现的行为：命中、无命中、大小写敏感、大小写不敏感、反选、反选与大小写叠加。最后那个尤其值得注意——`invert_and_ignore_case_together` 里的 `"Apple"` 因为大小写不敏感而被判为命中，于是被 `-v` **排除**，只剩 `apricot`。这是你能想到的最容易写错的组合，所以它值得一个测试。

**为什么测试要更新，而不是"测试本来就该过"？** 因为行号加入输出是**行为变更**，不是 bug。行为变了，契约描述就得跟着变——测试是契约的书面形式。第 1 部分我们说"测试把契约钉进代码"；现在补上后半句：**改了契约，要连着把钉子重新钉一遍**，而不是把测试删掉了事。

## 检查点

> [!PREDICT]
> 运行之前先承诺答案：`cargo run -- -v berry fruits.txt` 会打印几行？退出码是几？再预测一个边缘的：`cargo run -- -v "" fruits.txt` 会打印什么？（提示：空模式恒命中每一行，而 `-v` 要的正是"不命中"的行。）

```bash
cargo test
```

预期：`test result: ok. 6 passed`。

```bash
cargo run -- -v berry fruits.txt
echo "退出码：$?"
```

预期输出：

```
1: apple
2: apricot
3: banana
5: cherry
退出码：0
```

反选选中了四行——`blueberry` 是唯一被排除的。可打开系统 grep 对照：`grep -v -n berry fruits.txt`（`-n` 显示行号）。

```bash
cargo run -- -v mango fruits.txt
echo "退出码：$?"
```

预期：五行全部输出（没有一行含 `mango`，全部选中），`退出码：0`。

```bash
cargo run -- -v "" fruits.txt
echo "退出码：$?"
```

预期：**零输出**，`退出码：1`——空模式让每一行都算命中，`-v` 于是什么都不选，无选中行即 1。这个结果不是 bug，是契约在两处规则叠加下的自然产物。

```bash
cargo run -- -v
echo "退出码：$?"
```

预期：stderr 出现 `用法：grep-search [-v] [-i] <字面模式> <文件>`，`退出码：2`——只有开关、没有模式与文件，`rest.len()` 是 0。

可能的错误：

- 报 `cannot find function matching_lines`——旧测试或 `main` 里还有旧名字，按编译器提示逐个改到 `selected_lines`。
- 报 `if and else have incompatible types` 或 `temporary value dropped while borrowed`——练习 1 的老写法还留在某处，对照 `selected_lines` 里 `haystack`/`needle` 的写法改。
- 测试里报 `expected (usize, String)`——断言还写着 `vec!["banana"]`，元组与行号没加上。
- `-v` 参数被当成模式——`parse_args` 里 `-v` 的分支没写或拼错（区分 `-v` 与 `--invert-match`？我们只认 `-v`，这是练习 4 的范围）。

## 目标达成回顾

原定学习目标：**掌握用 Rust 从零构建命令行搜索工具的核心闭环——参数解析、逐行读取、字符串匹配、错误处理与退出码，并用测试守护行为。**

- **参数解析**：已覆盖并前进一步。第 1 部分用 `Vec<String>` + 个数判断；本部分升级为 `parse_args` 输出 `Options`，能容纳任意开关，且位置无关。
- **逐行读取**：已覆盖（`read_to_string` + `lines()`）。仍有一次性的取舍：整体读入内存，只支持 UTF-8。大文件与字节流尚未覆盖。
- **字符串匹配**：已覆盖字面匹配的正选与反选、大小写敏感与不敏感。**正则尚未覆盖**——它是最明显的下一步。
- **错误处理与退出码**：已完整覆盖三态（0/1/2），并验证了反选不改变规则本身；错误消息按种类分化（你的练习 3）。
- **测试守护行为**：已覆盖，并且本部分第一次演练了"行为变更 → 测试同步更新"的完整循环。

结论：**基本达成**。目标里的闭环已经闭合，且结构经受过一次真实的扩展考验。尚未纳入的有：正则、目录递归、stdin、大文件流式读取、`--help` 这类完整的选项体验。

接下来由你决定：继续延伸某个方向（`/lathe 续写 grep-search`），或者觉得这把工具已经够用、想就此收官（`/lathe 续写 收尾 grep-search`，会生成一份总结并标记课程完结）。

## 接下来

`-v` 之后，这把工具开始有一点点"能组装"的意思了：新选项只需在 `parse_args` 里加一个分支、在 `Options` 里加一个字段、在 `selected_lines` 里加一条判断，`main` 一个字都不用动。这个形状值钱——它是你后面加 `-n`、`-c`、`-r` 的地基。

还悬着的问题不少：模式只是字面文本（真实 grep 默认正则，`grep 'pan.*c'` 能匹配 `panic` 和 `pacific`）；只能搜一个文件；只能从文件读，不能从管道读；整个文件进内存。随便挑一个，都够撑起下一个部分。

在继续之前，用你自己的话回答一个问题：**为什么行号应该由 `selected_lines` 交出来，而 `"3: banana"` 这种字符串不该由它拼？** 提示：想一想如果哪天要加 `-c`（只输出计数）或 `--color`（高亮匹配片段），`selected_lines` 需不需要改。这不是"怎么写"的问题，是"谁该知道什么"的问题——答得出来，你就摸到了划分职责的那条线。

## 练习

1. **把行号变成开关 `-n`。** 现在行号总是打印。改成默认不打印、加了 `-n` 才打印（真实 grep 就是这个默认值）。`selected_lines` 不用改——它照旧交出 `(行号, 内容)`，变的只是 `main` 里那个 `println!`。这正好检验职责划分有没有做对。
2. **给 `parse_args` 写测试。** 它也是纯函数！试试 `parse_args(&["grep-search".to_string(), "-v".to_string(), "berry".to_string(), "fruits.txt".to_string()])`，断言 `invert == true`、`pattern == "berry"`。再测一个失败的：只给一个位置参数时，`is_err()` 应为真。注意 `Options` 的字段需要 `pub` 或测试与定义同模块——你现在的结构恰好满足后者。
3. **加 `--help`。** 当参数中出现 `-h` 或 `--help` 时，把用法打印到 **stdout** 并返回 `ExitCode::SUCCESS`（0）——工具主动帮你，是成功，不是错误。想想为什么这里走 stdout 而错误信息走 stderr。
4. **支持连写短开关 `-vi`。** 现在 `-vi` 会被当成未知参数塞进 `rest`，最终报"用法"。让它等价于 `-v -i`。提示：判断 `arg.starts_with('-')` 后逐字符匹配。做完顺手想想：`-vi` 与 `-v -i` 都能工作后，`parse_args` 里的 `rest` 判断还够用吗？

## 来源

**grep 行为**

1. [GNU Grep 3.12 手册 — Matching Control](https://www.gnu.org/software/grep/manual/html_node/Matching-Control.html) —— `-v, --invert-match` 的定义：反转匹配的意义。
2. [GNU Grep 3.12 手册 — Exit Status](https://www.gnu.org/software/grep/manual/html_node/Exit-Status.html) —— 退出码按"是否有行被选中"判定，`-v` 只改变"选中"的含义。

**Rust 参考**

3. [Rust 错误索引 — E0716: temporary value dropped while borrowed](https://doc.rust-lang.org/error_codes/E0716.html) —— 借用临时值的经典错误。
4. [The Rust Reference — Destructors: 临时作用域与延长规则](https://doc.rust-lang.org/reference/destructors.html#extending-based-on-expressions) —— 延长规则基于语法形式，`if`/`match` 分支后来才被纳入。
5. [clap derive 教程](https://docs.rs/clap/latest/clap/_derive/index.html) —— 你在 `todo-cli-rust` 里用过的 `#[derive(Parser)]`，与我们手写的 `parse_args` 做的是同一件事。
