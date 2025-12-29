# Шаблон локального backend-проекта

Шпаргалка для первичного развертывания необходимых инструментов и окружения

## Оглавление

1. [Homebrew](#homebrew)
2. [Homebrew formulae: CLI-утилиты](#homebrew-formulae-cli-утилиты)
3. [Homebrew casks: GUI-приложения](#homebrew-casks-gui-приложения)
4. [Oh My Zsh](#oh-my-zsh)
5. [Git](#git)
6. [GitHub SSH](#github-ssh)
7. [GitLab SSH](#gitlab-ssh)
8. [Python](#python)


## Homebrew


### Установка

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
>    - тянет официальный скрипт и вежливо раскладывает Homebrew в /opt/homebrew


### Подключение brew к zsh, чтобы пакеты Homebrew всегда были впереди системных

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```
>   - приписывает brew-переменные в профиль, чтобы PATH начинался с /opt/homebrew/bin

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```
>   - подхватывает эти переменные прямо сейчас, без выхода из терминала

```bash
exec zsh -l
```
>   - перезапускает оболочку, мгновенно «видя» новый PATH


### Скрипт проверки: «Краткий паспорт brew»

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

    local -i W_N=3 W_F=24 W_V=12 W_O=8 W_B=8
    local -i W_APP=32 W_A=12 W_C=44

    _trim () { local s="$1"; local -i w="$2"; s="${s//$'\n'/ }";
               (( ${#s} > w )) && print -r -- "${s[1,$((w-1))]}…" || print -r -- "$s"; }
    _line () { printf '─%.0s' {1..110}; echo; }

    echo "CLI header"
    _line
    printf "%*s %s %-*s %s %-*s %s %-*s %s %-*s\n" \
        $W_N "№" "$SEP" \
        $W_F "Пакет" "$SEP" \
        $W_V "Версия" "$SEP" \
        $W_O "Стрелка" "$SEP" \
        $W_B "Команда"
    _line

    local i=0 line f ver opt_col bin_col cellar
    while IFS= read -r line; do
        f="${line%% *}"
        ver="${line#* }"
        [[ -n "$filter" && "$f" != *"$filter"* && "$ver" != *"$filter"* ]] && continue

        ((i++))

        opt_col="—"
        [[ -L "$prefix/opt/$f" ]] && opt_col="symlink"

        bin_col="—"
        cellar="$prefix/Cellar/$f/$ver"
        [[ -d "$cellar/bin" && -n "$cellar/bin"/*(N) ]] && bin_col="есть"

        printf "%*s %s %-*s %s %-*s %s %-*s %s %-*s\n" \
            $W_N "$i" "$SEP" \
            $W_F "$(_trim "$f" $W_F)" "$SEP" \
            $W_V "$(_trim "$ver" $W_V)" "$SEP" \
            $W_O "$(_trim "$opt_col" $W_O)" "$SEP" \
            $W_B "$(_trim "$bin_col" $W_B)"
    done < <(brew list --formula --versions 2>/dev/null)

    echo

    echo "GUI header"
    _line
    printf "%*s %s %-*s %s %-*s %s %-*s\n" \
        $W_N   "№" "$SEP" \
        $W_APP "Приложение" "$SEP" \
        $W_A   "Тушка" "$SEP" \
        $W_C   "Стрелка"
    _line

    setopt local_options null_glob

    i=0
    local cask cver cdir app_path app_name a_col c_col
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

        a_col="—"
        [[ -d "/Applications/$app_name" ]] && a_col="bundle"
        [[ -L "/Applications/$app_name" ]] && a_col="symlink"

        c_col="—"
        if [[ -L "$app_path" ]]; then
            c_col="symlink → $(readlink "$app_path")"
        elif [[ -d "$app_path" ]]; then
            c_col="bundle"
        fi
        [[ ${#c_col} -gt $W_C ]] && c_col="${c_col[1,$((W_C-1))]}…"

        printf "%*s %s %-*s %s %-*s %s %-*s\n" \
            $W_N "$i" "$SEP" \
            $W_APP "$(_trim "$app_name" $W_APP)" "$SEP" \
            $W_A   "$(_trim "$a_col" $W_A)" "$SEP" \
            $W_C   "$c_col"
    done < <(brew list --cask --versions 2>/dev/null)
}

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

### Проверка

```bash
which brew && brew -v && brew doctor
```
>   - убеждаемся, что brew в PATH, печатаем версию и устраиваем системный медосмотр


### Обновление

```bash
brew update && brew upgrade && brew cleanup
```
>   - подтягиваем свежие рецепты, обновляем бутылки и выбрасываем залежалый кэш


### Деинсталляция

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"
```
>   - аккуратно вычищает Homebrew и всё, что он установил, оставляя macOS в первозданном виде


## Homebrew formulae: CLI-утилиты


### Установка

```bash
brew search <имя>
```
>    «маячим фонариком» — ищем нужную формулу в каталоге

```bash
brew install <имя>
```
>    команда-телепорт: тянет бинарь и ставит его в /opt/homebrew


### Проверка

```bash
which <binary> && <binary> --version
```
>    показываем путь к бинарю и убеждаемся, что он живой и подписан


### Обновление

```bash
brew update && brew upgrade <имя> && brew cleanup <имя>
```
>    освежаем рецепты, апгрейдим утилиту и метём следы старых версий


### Деинсталляция

```bash
brew uninstall <имя> && brew cleanup <имя>
```
>    выкорчёвываем пакет и сметаем оставшиеся «бутылки»


## Homebrew casks: GUI-приложения


### Установка

```bash
brew search --cask <имя>
```
>    осматриваем «выставку» приложений в разделе cask

```bash
brew install --cask <имя>
```
>    скачиваем .app, подписываем и кладём в /Applications

```bash
brew install --cask --adopt <имя>
```
>    «усыновляем» .app, уже лежащие в /Applications, и переводим их под апдейт-робота brew


### Проверка

```bash
brew list --cask | grep <имя>
```
>    быстрая сверка: приложение числится в списке?

```bash
ls -l /Applications/<имя>.app
ls -l /opt/homebrew/Caskroom/<имя>/*/<имя>.app
```
>    где «тушка», где стрелка? c конца 2022 г. Homebrew держит bundle в /Applications, а сим-линк — в Caskroom


### Обновление

```bash
brew update && brew upgrade --cask <имя> && brew cleanup --cask <имя>
```
>    подтягиваем свежий .app и чистим архивы

```bash
brew update && brew upgrade --cask && brew cleanup --cask
```
>    массовая стирка: обновляем ВСЕ GUI-приложения одним махом


### Деинсталляция

```bash
brew uninstall --cask <имя> && brew cleanup --cask <имя>
```
>    удаляем .app, квитанцию и кэш, словно её никогда не было

```bash
brew uninstall --cask --zap <имя> && brew cleanup --cask <имя>
```
>    то же, но дополнительно чистим данные в ~/Library


## Oh My Zsh


### Установка

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```
>    волшебный скрипт ставит Oh My Zsh и создаёт базовый ~/.zshrc


#### Тема Powerlevel10k

```bash
    brew install --cask font-meslo-lg-nerd-font
```
>    шрифт-мультииконсет, чтобы значки выглядели круто

```bash
    brew install powerlevel10k
```
>    тема-ракета для Zsh с мгновенным откликом

```bash
    echo 'source $(brew --prefix)/share/powerlevel10k/powerlevel10k.zsh-theme' >> ~/.zshrc
```
>    подключаем тему в конфиг

```bash
    p10k configure
```
>    запускает интерактивный мастер настройки


#### Полезные плагины

```bash
    git clone https://github.com/zsh-users/zsh-autosuggestions \
        ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```
>    автодополнение команд «на лету»

```bash
    git clone https://github.com/zsh-users/zsh-syntax-highlighting \
        ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```
>    подсвечивает синтаксис прямо в командной строке


#### затем пропишите плагины в ~/.zshrc: plugins=(git zsh-autosuggestions zsh-syntax-highlighting)



### Проверка

```bash
zsh --version && echo $ZSH
```
>    видим версию Zsh и путь до Oh My Zsh — значит всё подключилось


### Обновление

```bash
zstyle ':omz:update' mode reminder
```
>    раз в две недели Oh My Zsh вежливо напомнит обновиться

```bash
omz update
```
>    апдейтит фреймворк, плагины и темы за один проход


### Деинсталляция

```bash
rm -rf ~/.oh-my-zsh{,_custom} ~/.p10k.zsh ~/.zcompdump* ~/.zshrc ~/.zprofile ~/.zsh_history 2>/dev/null
```
>    чистим фреймворк, темы и конфиг до нуля

```bash
chsh -s /bin/zsh
```
>    возвращаемся к «чистому» системному Zsh без наворотов


## Git


### Установка

```bash
brew install git
```
>    ставим свежайший Git, чтобы не зависеть от системной версии


### Проверка

```bash
which git && git -v
```
>    убеждаемся, что бинарь в /usr/local/bin и версия та, что надо


### Обновление

```bash
brew update && brew upgrade git && brew cleanup git
```
>    подтягиваем новые формулы, обновляем Git и чистим хвосты


### Деинсталляция

```bash
brew uninstall git && brew cleanup git
```
>    удаляем Git из Homebrew-песочницы и подчистим кэш


## GitHub SSH


### Генерация ключа

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```
>    создаём современный ED25519-ключ для шифрованных push’ей

```bash
eval "$(ssh-agent -s)"
```
>    поднимаем ssh-агента, чтобы он хранил ключ в памяти

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```
>    добавляем приватный ключ и просим macOS помнить пароль


### Добавление в GitHub

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```
>    кидаем публичный ключ в буфер, чтобы вставить в «SSH Keys»

```bash
# Переходим в GitHub → Settings → SSH and GPG keys → New SSH key
```
>    вставляем скопированный ключ и сохраняем


### Проверка подключения

```bash
ssh -T git@github.com
```
>    GitHub должен дружелюбно поздороваться по SSH


### Обновление ключа

```bash
ssh-keygen -R github.com && ssh-keyscan github.com >> ~/.ssh/known_hosts
```
>    обнуляем устаревший fingerprint и заново кэшируем свежий


### Деинсталляция ключа

```bash
rm ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```
>    убираем ключи с диска (не забудьте удалить из GitHub)


## GitLab SSH


### Генерация ключа

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```
>    генерим отдельный ключ или используем уже существующий


### Добавление в GitLab

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```
>    копируем паблик в клипборд

```bash
# GitLab → Preferences → SSH Keys → Add SSH Key
```
>    вставляем, задаём срок действия при желании и сохраняем


### Проверка подключения

```bash
ssh -T git@gitlab.com
```
>    GitLab ответит приветствием, если всё ок


### Обновление ключа

```bash
ssh-keygen -R gitlab.com && ssh-keyscan gitlab.com >> ~/.ssh/known_hosts
```
>    если GitLab сменил fingerprint, обновляем кэш вручную


### Деинсталляция ключа

```bash
rm ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```
>    стираем ключи локально и удаляем их из GitLab-профиля


## Python


### Установка

```bash
brew search python@
```
>    смотрим, какие ветки Python доступны в репозитории

```bash
brew install python@3.x
```
>    ставим нужную ветку, не трогая системный Python


### Быстрое переключение «главной» версии (сим-линк python3)

```bash
brew unlink python@3.11
```
>    отвязываем текущую ветку из /opt/homebrew/opt

```bash
brew link --overwrite --force python@3.12
```
>    назначаем 3.12 основной; операция <1 с-и, данных не копирует


### Подключаем Homebrew к zsh, чтобы brew-пакеты шли раньше системных

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```
>    дописываем переменные среды в профиль раз и навсегда

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```
>    подхватываем переменные прямо в текущей сессии

```bash
exec zsh -l
```
>    перезапускаем оболочку, чтобы PATH обновился


### Проверка

```bash
which python3 && python3 --version && pip3 --version
```
>    убедились, что бинари указывают на /opt/homebrew и версии свежие


### Обновление

```bash
brew update && brew upgrade python && brew cleanup python
```
>    подтягиваем формулы, обновляем сам Python и чистим старые бутылки


### Деинсталляция

```bash
brew uninstall python && brew cleanup python
```
>    удаляем Python целиком и подчистим всякий кэш


#### для конкретной ветки:

```bash
    brew uninstall python@3.x
```
>    точечно сносим установленную версию
