# Шаблон backend-проекта

Шпаргалка для первичного развертывания необходимых инструментов

## Оглавление

1. [Homebrew](#homebrew)
2. [Homebrew formulae](#homebrew-formulae)
3. [Homebrew casks](#homebrew-casks)
4. [Oh My Zsh](#oh-my-zsh)
5. [Git](#git)
6. [GitHub SSH](#github-ssh)
7. [GitLab SSH](#gitlab-ssh)
8. [Python](#python)
9. [Venv](#venv)
10. [Poetry](#poetry)


## Homebrew
`Пакетный менеджер для установки CLI и GUI инструментов`


### Установить

> Установить Homebrew через официальный install.sh

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

> Подключить Homebrew к zsh, чтобы пакеты Homebrew были впереди системных.

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```

> Применить shellenv в текущей сессии терминала.

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

> Перезапустить zsh, чтобы обновления окружения применились.

```bash
exec zsh -l
```

### Проверить

> Проверить доступность Homebrew в PATH и запустить диагностику.

```bash
which brew && brew -v && brew doctor
```

> Добавить скрипт проверки: «Краткий паспорт brew».

```bash
# 1. Добавить
cat >> ~/.zshrc <<'EOF'
# Скрипт проверки: «Краткий паспорт brew»
# Запуск: brew_where  |  brew_where <formula or cask> | brew_where_one <formula>
brew_where () {
    local prefix filter SEP
    prefix="$(brew --prefix)"
    filter="$1"
    SEP="│"

    local -i W_N=3 W_NAME=24 W_VER=14 W_LOC=14 W_LINK=44

    _trim () { local s="$1"; local -i w="$2"; s="${s//$'\n'/ }";
               (( ${#s} > w )) && print -r -- "${s[1,$((w-1))]}…" || print -r -- "$s"; }
    _line () { printf '─%.0s' {1..125}; echo; }

    # Таблица CLI header
    echo "CLI header"
    _line
    printf "%*s %s %-*s %s %-*s %s %-*s %s %-*s\n" \
        $W_N    "№" "$SEP" \
        $W_NAME "Пакет" "$SEP" \
        $W_VER  "Версия" "$SEP" \
        $W_LOC  "/Cellar" "$SEP" \
        $W_LINK "/opt"
    _line

    local i=0 line f ver opt_col bin_col cellar
    while IFS= read -r line; do
        f="${line%% *}"
        ver="${line#* }"
        [[ -n "$filter" && "$f" != *"$filter"* && "$ver" != *"$filter"* ]] && continue

        ((i++))

        cellar="$prefix/Cellar/$f/$ver"

        bin_col="—"
        [[ -d "$cellar" ]] && bin_col="bundle"

        opt_col="—"
        [[ -L "$prefix/opt/$f" ]] && opt_col="symlink"

        printf "%*s %s %-*s %s %-*s %s %-*s %s %-*s\n" \
            $W_N    "$i" "$SEP" \
            $W_NAME "$(_trim "$f" $W_NAME)" "$SEP" \
            $W_VER  "$(_trim "$ver" $W_VER)" "$SEP" \
            $W_LOC  "$(_trim "$bin_col" $W_LOC)" "$SEP" \
            $W_LINK "$(_trim "$opt_col" $W_LINK)"
    done < <(brew list --formula --versions 2>/dev/null)

    echo

    # Таблица GUI header
    echo "GUI header"
    _line
    printf "%*s %s %-*s %s %-*s %s %-*s %s %-*s\n" \
        $W_N    "№" "$SEP" \
        $W_NAME "Приложение" "$SEP" \
        $W_VER  "Версия" "$SEP" \
        $W_LOC  "/Applications" "$SEP" \
        $W_LINK "/Caskroom"
    _line

    setopt local_options null_glob

    i=0
    local cask cver cdir app_path app_name v_col a_col c_col
    while IFS= read -r line; do
        cask="${line%% *}"
        cver="${line#* }"
        [[ -n "$filter" && "$cask" != *"$filter"* ]] && continue

        cdir="$prefix/Caskroom/$cask/$cver"
        [[ ! -d "$cdir" ]] && continue

        app_path="$(find "$cdir" -maxdepth 3 -name '*.app' -print -quit 2>/dev/null)"
        [[ -z "$app_path" ]] && continue

        app_name="${app_path:t}"
        [[ -n "$filter" && "$app_name" != *"$filter"* && "$cask" != *"$filter"* ]] && continue

        ((i++))

        v_col="$cver"

        a_col="—"
        [[ -d "/Applications/$app_name" ]] && a_col="bundle"
        [[ -L "/Applications/$app_name" ]] && a_col="symlink"

        c_col="—"
        if [[ -L "$app_path" ]]; then
            c_col="symlink"
        elif [[ -d "$app_path" ]]; then
            c_col="bundle"
        fi
        [[ ${#c_col} -gt $W_LINK ]] && c_col="${c_col[1,$((W_LINK-1))]}…"

        printf "%*s %s %-*s %s %-*s %s %-*s %s %-*s\n" \
            $W_N    "$i" "$SEP" \
            $W_NAME "$(_trim "$app_name" $W_NAME)" "$SEP" \
            $W_VER  "$(_trim "$v_col" $W_VER)" "$SEP" \
            $W_LOC  "$(_trim "$a_col" $W_LOC)" "$SEP" \
            $W_LINK "$c_col"
    done < <(brew list --cask --versions 2>/dev/null)
}

# Подробности по формуле (когда нужно, а где тушка и какие команды)
brew_where_one () {
    local f="$1" prefix ver cellar
    [[ -z "$f" ]] && { echo "Использование: brew_where_one <formula>"; return 1; }
    prefix="$(brew --prefix)"
    ver="$(brew list --versions "$f" 2>/dev/null | awk '{print $2}')"
    [[ -z "$ver" ]] && { echo "Формула не установлена: $f"; return 1; }

    cellar="$prefix/Cellar/$f/$ver"
    echo "FORMULA: $f $ver"
    echo "opt:     $prefix/opt/$f"
    [[ -L "$prefix/opt/$f" ]] && echo "         -> $(readlink "$prefix/opt/$f")"
    echo "cellar:  $cellar"
    echo "bin:"
    ls -1 "$cellar/bin" 2>/dev/null | sed 's/^/  - /' || echo "  —"
}
EOF

# 2. Применить
source ~/.zshrc
echo "✅ Готово. 🚀 Запуск: brew_where  |  brew_where <formula or cask> | brew_where_one <formula>"
```

### Обновить

> Обновить формулы, очистить устаревшие версии и кеш.

```bash
brew update && brew upgrade && brew cleanup
```

### Удалить

> Удалить Homebrew через официальный uninstall.sh.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"
```

## Homebrew formulae
`CLI-утилиты`


### Установить

> Найти формулу в каталоге Homebrew.

```bash
brew search <имя>
```

> Установить формулу через Homebrew.

```bash
brew install <имя>
```

### Проверить

> Проверить путь к бинарю и вывести версию.

```bash
which <binary> && <binary> --version
```

### Обновить

> Обновить формулу и удалить старые версии.

```bash
brew update && brew upgrade <имя> && brew cleanup <имя>
```

### Удалить

> Удалить формулу и очистить оставшиеся артефакты.

```bash
brew uninstall <имя> && brew cleanup <имя>
```

## Homebrew casks
`GUI-приложения`


### Установить

> Найти cask-приложение в каталоге Homebrew.

```bash
brew search --cask <имя>
```

> Установить GUI-приложение как cask, обычно размещается в /Applications.

```bash
brew install --cask <имя>
```

> Привязать уже установленное приложение к Homebrew, чтобы обновлять его через Homebrew.

```bash
brew install --cask --adopt <имя>
```

### Проверить

> Проверить, что cask установлен.

```bash
brew list --cask | grep <имя>
```

> Показать, где лежит приложение и как оно связано с Caskroom.

```bash
ls -l /Applications/<имя>.app
ls -l /opt/homebrew/Caskroom/<имя>/*/<имя>.app
```

### Обновить

> Обновить выбранный cask и очистить старые версии.

```bash
brew update && brew upgrade --cask <имя> && brew cleanup <имя>
```

> Обновить все cask-приложения и очистить старые версии.

```bash
brew update && brew upgrade --cask && brew cleanup
```

### Удалить

> Удалить cask-приложение и очистить связанные артефакты.

```bash
brew uninstall --cask <имя> && brew cleanup <имя>
```

> Удалить cask-приложение и дополнительно очистить пользовательские данные.

```bash
brew uninstall --cask --zap <имя> && brew cleanup <имя>
```

## Oh My Zsh
`Оболочка zsh, темы и плагины для удобной работы с терминалом`


### Установить

> Установить Oh My Zsh и создать базовый ~/.zshrc.

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

#### Установить тему Powerlevel10k

> Установить Nerd Font, чтобы тема корректно показывала символы.

```bash
    brew install --cask font-meslo-lg-nerd-font
```

> Установить тему powerlevel10k.

```bash
    brew install powerlevel10k
```

> Подключить тему powerlevel10k в ~/.zshrc.

```bash
    echo 'source $(brew --prefix)/share/powerlevel10k/powerlevel10k.zsh-theme' >> ~/.zshrc
```

> Запустить мастер настройки powerlevel10k.

```bash
    p10k configure
```

#### Установить полезные плагины

> Установить плагин подсветки синтаксиса.

```bash
    git clone https://github.com/zsh-users/zsh-syntax-highlighting \
        ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

> Установить плагин автоподсказок команд.

```bash
    git clone https://github.com/zsh-users/zsh-autosuggestions \
        ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

> Установить плагин автоподсказок директорий.

```bash
    git clone https://github.com/marlonrichert/zsh-autocomplete \
        ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autocomplete
```

#### Включить плагины в .zshrc: plugins=(git zsh-syntax-highlighting zsh-autosuggestions zsh-autocomplete)


### Проверить

> Проверить версию zsh и наличие пути до Oh My Zsh $ZSH.

```bash
zsh --version && echo $ZSH
```

### Обновить

> Включить режим напоминаний об обновлении.

```bash
zstyle ':omz:update' mode reminder
```

> Обновить Oh My Zsh, темы и плагины, если подключены.

```bash
omz update
```

### Удалить

> Удалить Oh My Zsh и связанные файлы конфигурации.

```bash
rm -rf ~/.oh-my-zsh{,_custom} ~/.p10k.zsh ~/.zcompdump* ~/.zshrc ~/.zprofile ~/.zsh_history 2>/dev/null
```

> Вернуть системный /bin/zsh как оболочку по умолчанию.

```bash
chsh -s /bin/zsh
```

## Git
`Система контроля версий`


### Установить

> Установить Git через Homebrew.

```bash
brew install git
```

### Проверить

> Проверить путь к git и вывести версию.

```bash
which git && git -v
```

### Обновить

> Обновить Git и очистить старые версии.

```bash
brew update && brew upgrade git && brew cleanup git
```

### Удалить

> Удалить Git и очистить артефакты.

```bash
brew uninstall git && brew cleanup git
```

## GitHub SSH
`SSH-доступ к GitHub без пароля`


### Создать ключ

> Создать SSH-ключ ED25519.

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

> Запустить ssh-agent.

```bash
eval "$(ssh-agent -s)"
```

> Добавить ключ в ssh-agent и Keychain macOS.

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

### Добавить в GitHub

> Скопировать публичный ключ в буфер обмена.

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

> Добавить ключ в GitHub: Settings → SSH and GPG keys → New SSH key.

```bash
# Переходим в GitHub → Settings → SSH and GPG keys → New SSH key
```

### Проверить подключение

> Проверить SSH-подключение к GitHub.

```bash
ssh -T git@github.com
```

### Обновить ключ

> Обновить known_hosts для github.com.

```bash
ssh-keygen -R github.com && ssh-keyscan github.com >> ~/.ssh/known_hosts
```

### Удалить ключ

> Удалить ключи с диска, предварительно удалить их из GitHub.

```bash
rm ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

## GitLab SSH
`SSH-доступ к GitLab без пароля`


### Создать ключ

> Создать SSH-ключ ED25519.

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

### Добавить в GitLab

> Скопировать публичный ключ в буфер обмена.

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

> Добавить ключ в GitLab: Preferences → SSH Keys → Add SSH Key.

```bash
# GitLab → Preferences → SSH Keys → Add SSH Key
```

### Проверить подключение

> Проверить SSH-подключение к GitLab.

```bash
ssh -T git@gitlab.com
```

### Обновить ключ

> Обновить known_hosts для gitlab.com.

```bash
ssh-keygen -R gitlab.com && ssh-keyscan gitlab.com >> ~/.ssh/known_hosts
```

### Удалить ключ

> Удалить ключи с диска, предварительно удалить их из GitLab.

```bash
rm ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```




## Python
`Интерпретатор Python`


### Установить

> Показать доступные версии python@3.x в Homebrew.

```bash
brew search python@
```

> Установить выбранную версию Python через Homebrew.

```bash
brew install python@3.x
```

> Отвязать текущую версию python@3.11, убрать симлинки.

```bash
brew unlink python@3.11
```

> Сделать указанную версию python@3.12 основной, пересоздать симлинки.

```bash
brew link --overwrite --force python@3.12
```

### Проверить

> Проверить путь, вывести версии python3 и pip3.

```bash
which python3 && python3 --version && pip3 --version
```

### Обновить

> Обновить Python и очистить старые версии.

```bash
brew update && brew upgrade python && brew cleanup python
```

### Удалить

> Удалить Python и очистить артефакты.

```bash
brew uninstall python && brew cleanup python
```

> Удалить конкретную версию.

```bash
    brew uninstall python@3.x
```

## Venv
`Виртуальное окружение`


### Создать окружение

> Создать виртуальное окружение в папке проекта.

```bash
python3 -m venv .venv
```

> Добавить папку окружения в .gitignore, чтобы не коммитить окружение.

```bash
echo '.venv/' >> .gitignore
```

> Обновить pip внутри окружения.

```bash
.venv/bin/python -m pip install --upgrade pip
```


### Активировать окружение

> Активировать окружение для текущей сессии терминала.

```bash
source .venv/bin/activate
```


### Проверить

> Проверить, что python и pip идут из .venv.

```bash
which python
python --version
which pip
pip --version
```

> Проверить переменную активного окружения.

```bash
echo $VIRTUAL_ENV
```

### Установить и зафиксировать зависимости

> Установить пакет.

```bash
pip install <package>
```

> Зафиксировать зависимости в requirements.txt.

```bash
pip freeze > requirements.txt
```

> Установить зависимости из requirements.txt.

```bash
pip install -r requirements.txt
```

### Деактивировать окружение

> Выйти из окружения и вернуться к системному Python.

```bash
deactivate
```

### Удалить окружение

> Удалить папку окружения.

```bash
rm -rf .venv
```

## Poetry

## Ollama
`Описание`


### Установить
Здесь установка через ollama xthtp brew
Установка моделей gemma3:12b qwen2.5-coder через brew
Установка и связь с continue
Копирование yaml проектного в системный

### Проверить

### Обновить

###

## LM Studio
