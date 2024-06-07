---
title: 現時点での私のシェル＆プロンプト設定を晒す（というか自分の設定備忘録）
date: 2022-10-27
slug: my-shell-prompt-2022
tags:
  - alactitty
  - tmux
  - zsh
  - starship
---

## 前提

-   macOS Montrey
-   Homebrew 導入済み

## 使うもの

### Alacritty（ターミナルエミュレータ）

[公式サイトはこちら](https://github.com/alacritty/alacritty)。  
Rust 製の軽量・シンプルなターミナルエミュレータです。  
日本語入力がおかしいという問題があったりするものの（[参考](https://komi.dev/post/2021-07-20-enabling-ime-in-alacritty/)）、早くて使いやすいです。  
多機能なやつも試してみましたが、結局はここに落ち着きました。  
Homebrew 経由でインストール可能です：

```
$ brew install --cask alacritty
```

設定ファイルは `.config/alacritty/alacritty.yml` に置きます。[こちら](https://zenn.dev/kehra/articles/3cb06e5de83cc9)を参考に以下のように設定しています（次に紹介する tmux と関連した設定も入っています）：

```
window:
  # デカめがいいので
  dimensions:
    columns: 220
    lines: 75

  # macは角丸ウインドウなので余白をとったほうが良い
  padding:
    x: 8
    y: 4

  # 少し透過
  opacity: 0.85

scrolling:
  # consoleのlogを10000行まで保持
  history: 10000

  # スクロール量は3行
  multiplier: 3

# フォント設定（白源フォントを利用）
font:
  normal:
    family: 'HackGen Console NFJ'
    style: Regular

  bold:
    family: 'HackGen Console NFJ'
    style: Bold

  size: 14.0

  offset:
    y: 1

# 基本はデフォルトのzshだが、tmuxを起動するように設定する
# また起動済みのtmuxがあればattachにする
shell:
  program: /usr/local/bin/zsh
  args:
    - -l
    - -c
    - "/usr/local/bin/tmux a -t 0 || /usr/local/bin/tmux -u"

# キーバインド
key_bindings:
  # wikiのrecommnedをそのままコピーしただけ
  - { key: Comma,     mods: Command,      command:
      {program: "sh", args: ["-c","open ~/.config/alacritty/alacritty.yml"]}     }
  - { key: N,         mods: Command,      action: SpawnNewInstance        }
  - { key: Space,     mods: Alt,          chars: " "                      }
  - { key: Back,      mods: Super,   chars: "\x15"                        } # delete word/line
  - { key: Left,      mods: Alt,     chars: "\x1bb"                       } # one word left
  - { key: Right,     mods: Alt,     chars: "\x1bf"                       } # one word right
  - { key: Left,      mods: Command, chars: "\x1bOH",   mode: AppCursor   } # Home
  - { key: Right,     mods: Command, chars: "\x1bOF",   mode: AppCursor   } # End
  # tmuxのprefixをCtrl-Qにしているので、その設定
  # これがないとtmuxのprefixが効かずに、Alacrittyのキーバインドに持っていかれるっぽい？
  - { key: Q,         mods: Control, chars: "\x11"                        } # tmux prefix

# Colors (Solarized Dark)
colors:
  # Default colors
  primary:
    background: '#002b36' # base03
    foreground: '#839496' # base0

  # Cursor colors
  cursor:
    text:   '#002b36' # base03
    cursor: '#839496' # base0

  # Normal colors
  normal:
    black:   '#073642' # base02
    red:     '#dc322f' # red
    green:   '#859900' # green
    yellow:  '#b58900' # yellow
    blue:    '#268bd2' # blue
    magenta: '#d33682' # magenta
    cyan:    '#2aa198' # cyan
    white:   '#eee8d5' # base2

  # Bright colors
  bright:
    black:   '#002b36' # base03
    red:     '#cb4b16' # orange
    green:   '#586e75' # base01
    yellow:  '#657b83' # base00
    blue:    '#839496' # base0
    magenta: '#6c71c4' # violet
    cyan:    '#93a1a1' # base1
    white:   '#fdf6e3' # base3
```

### tmux（端末多重化ソフトウェア）

Alacritty は iTerm2 のようにタブ機能がついてないので、1つのターミナルを画面分割などをすることで複数のターミナルとして操作できるようにするために導入します。  
こちらも、Homebrew 経由でインストールできます：

```
$ brew install tmux
```

設定ファイルは `~/.config/tmux.conf` に置きます。

```
# tmux起動時のシェルをzshにする
set-option -g default-shell /bin/zsh

# tmuxを256色表示できるようにする
set-option -g default-terminal screen-256color
set -g terminal-overrides 'xterm:colors=256'

# prefixキーをC-qに変更
set -g prefix C-q

# C-bのキーバインドを解除
unbind C-b

# ステータスバーをトップに配置する
set-option -g status-position top

# 左右のステータスバーの長さを決定する
set-option -g status-left-length 90
set-option -g status-right-length 90

# #P => ペイン番号
# 最左に表示
set-option -g status-left '#H:[#P]'

# Wi-Fi、バッテリー残量、現在時刻
# 最右に表示
set-option -g status-right '#(wifi) #(battery --tmux) [%Y-%m-%d(%a) %H:%M]'

# ステータスバーを1秒毎に描画し直す
set-option -g status-interval 1

# センタライズ（主にウィンドウ番号など）
set-option -g status-justify centre

# ステータスバーの色を設定する
set-option -g status-bg "colour238"

# status line の文字色を指定する。
set-option -g status-fg "colour255"

# vimのキーバインドでペインを移動する
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# vimのキーバインドでペインをリサイズする
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5

# | でペインを縦分割する
bind | split-window -h

# - でペインを縦分割する
bind - split-window -v

# 番号基準値を変更
set-option -g base-index 1

# マウス操作を有効にする
set-option -g mouse on
bind -n WheelUpPane if-shell -F -t = "#{mouse_any_flag}" "send-keys -M" "if -Ft= '#{pane_in_mode}' 'send-keys -M' 'copy-mode -e'"

# コピーモードを設定する
# コピーモードでvimキーバインドを使う
setw -g mode-keys vi

# 'v' で選択を始める
bind -T copy-mode-vi v send -X begin-selection

# 'V' で行選択
bind -T copy-mode-vi V send -X select-line

# 'C-v' で矩形選択
bind -T copy-mode-vi C-v send -X rectangle-toggle

# 'y' でヤンク
bind -T copy-mode-vi y send -X copy-selection

# 'Y' で行ヤンク
bind -T copy-mode-vi Y send -X copy-line

# 'C-p'でペースト
bind-key C-p paste-buffer
```

### Zsh（シェル）

[fish shell](https://fishshell.com/) とか [Nushell](https://github.com/nushell/nushell) のような新しいシェルも試してみましたが、結局 zsh に戻ってきました。  
mac に標準で最初から入っているものを使っています。  
Nushell は Rust 製なので気にはなってます。もうちょっと使いやすくなったら乗り換えるかも。

### Starship（プロンプト）

[公式サイトはこちら](https://starship.rs/ja-JP/)。  
今まで Zsh の設定をゴリゴリ書いたり、[oh-my-zsh](https://ohmyz.sh/) を利用したりして、プロンプトのカスタマイズをしてたりしましたが、これで一発で解決しました。Rust 製です。  
これもまた、Homebrew 経由でインストールできます：

```
$ brew install starship
```

設定ファイルは `~/.config/starship.toml` で可能です。[こちら](https://zenn.dev/oosuke3/scraps/38a42f48c6dba4)を参考に設定していきます：  
　（もっと高度な設定は[公式ドキュメント](https://starship.rs/ja-JP/config/#%E3%83%95%E3%82%9A%E3%83%AD%E3%83%B3%E3%83%95%E3%82%9A%E3%83%88)を参考にすればできそうですが、追々で…）

```
add_newline = true

[character]
success_symbol = "[>](bold green)"
error_symbol = "[✗](bold red)"

[username]
style_user = "white bold"
style_root = "black bold"
format = "user: [$user]($style) "
disabled = false
show_always = true

[directory]
truncation_length = 100
truncate_to_repo = false

[memory_usage]
disabled = false
threshold = -1
style = "bold dimmed green"
format = "via [${ram_pct} / ${swap_pct}]($style) "

[package]
disabled = true

[time]
disabled = false
format = '[\[ $time \]]($style) '
```

### 白源フォント

[公式サイトはこちら](https://github.com/yuru7/HackGen)。  
[Ricty](https://rictyfonts.github.io/) が好きでずっと使ってきたんけど、最近はこちらのフォントがお気に入りです。  
Starship には [Nerd Font](https://www.nerdfonts.com/) が必要になりますが、白源フォントにも Nerd Font を追加した版があります。ダウンロードは Homebrew 経由でもできますが、[GitHub リポジトリのリリースページ](https://github.com/yuru7/HackGen/releases)から直接 ttf ファイルをダウンロードしてインストールできます。

### zplug

zsh のプラグインマネージャーです。Homebrew経由で導入可能です：

```
$ brew install zplug
```

以下の通り、`~/.zshrc` の設定を書き、有用なプラグインをいくつか入れておきます：

```
export ZPLUG_HOME=$(brew --prefix)/opt/zplug
source $ZPLUG_HOME/init.zsh

# 構文のハイライト(https://github.com/zsh-users/zsh-syntax-highlighting)
zplug "zsh-users/zsh-syntax-highlighting", defer:2
# 過去に入力したコマンドの履歴が灰色のサジェストで出る
zplug "zsh-users/zsh-history-substring-search"
zplug "zsh-users/zsh-autosuggestions", defer:2
# 補完強化
zplug "zsh-users/zsh-completions"
# コマンド入力途中で上下キー押したときの過去履歴がいい感じに出るようになる
zplug "zsh-users/zsh-history-substring-search"
# 256色表示にする
zplug "chrissicool/zsh-256color"

# インストールしていないプラグインがあればインストールする
if ! zplug check --verbose; then
    printf "Install? [y/N]: "
    if read -q; then
        echo; zplug install
    fi
fi
```

### exa（`ls` コマンドの拡張 ）

[公式サイトはこちら](https://github.com/ogham/exa)。  
Rust 製の `ls` コマンドの拡張コマンドです。  
Homebrew で導入可能です：

```
$ brew install exa
```

`~/.zshrc` に以下のようにエイリアスを定義しておき、 `ls` を打った時など、 exa を呼び出すようにします：

```
if [[ $(command -v exa) ]]; then
  alias e='exa --icons --git'
  alias l=e
  alias ls=e
  alias ea='exa -a --icons --git'
  alias la=ea
  alias ee='exa -aahl --icons --git'
  alias ll=ee
  alias et='exa -T -L 3 -a -I "node_modules|.git|.cache" --icons'
  alias lt=et
  alias eta='exa -T -a -I "node_modules|.git|.cache" --color=always --icons | less -r'
  alias lta=eta
  alias l='clear && ls'
fi
```

## 設定まとめ

[こちら](https://github.com/tsuchinoko0402/dotfiles/tree/main/home)に今回の設定+αをまとめています。

## 所感

やっぱ、Rust製のツールは応援したくなるよね。