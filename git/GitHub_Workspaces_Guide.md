# GitHub Workspaces: несколько GitHub-аккаунтов на одном Mac

> Практический гайд: отдельные SSH-ключи, Git identity и автоматическое подключение конфигураций по расположению репозитория.

Если у вас несколько GitHub-аккаунтов — личный, корпоративный и клиентский — не нужно постоянно переключать пользователя вручную. Настройте **GitHub Workspaces**: у каждого каталога проектов будет своя Git-конфигурация и SSH-ключ.

![Общая архитектура](overview.png)

## 1. Что разделяем

![Три уровня](levels.png)

- **Workspace** — директория, по которой `includeIf` выбирает настройки Git.
- **Git identity** (`user.name`, `user.email`) — автор коммита; **не** определяет права доступа.
- **SSH authentication** — ключ, которым GitHub аутентифицирует подключение; права зависят от аккаунта и репозитория.

Это **логическая**, а не системная изоляция: один пользователь macOS всё ещё имеет доступ к обоим ключам. GitHub CLI, HTTPS и браузер имеют отдельную авторизацию.

## 2. Структура каталогов

```text
~/
├── .gitconfig                     # центральные правила
├── git-config/
│   ├── work.gitconfig             # личный аккаунт iDEVconn
│   └── itcampus.gitconfig         # корпоративный аккаунт
├── .ssh/
│   ├── config                     # SSH-алиасы
│   ├── id_ed25519_idevconn        # приватный ключ
│   ├── id_ed25519_idevconn.pub    # публичный ключ
│   ├── github_itcampusdesc
│   └── github_itcampusdesc.pub
└── projects/
    ├── work/
    │   └── my-personal-repo/
    └── itcampus/
        └── my-company-repo/
```

Создайте каталоги (или подставьте существующие пути):

```bash
mkdir -p ~/git-config ~/projects/work ~/projects/itcampus
```

## 3. Проверка и создание SSH-ключей

Сначала проверьте существующие ключи:

```bash
ls -la ~/.ssh/
cat ~/.ssh/config
```

**Не перезаписывайте существующие приватные ключи.** Только если ключей ещё нет, создайте:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_idevconn -C "YOUR_PERSONAL_EMAIL"
ssh-keygen -t ed25519 -f ~/.ssh/github_itcampusdesc -C "YOUR_WORK_EMAIL"
```

Зарегистрируйте каждый **публичный** ключ в соответствующем GitHub-аккаунте: [GitHub SSH keys](https://github.com/settings/keys). Для копирования на Mac:

```bash
pbcopy < ~/.ssh/id_ed25519_idevconn.pub
# После регистрации первого ключа:
pbcopy < ~/.ssh/github_itcampusdesc.pub
```

## 4. SSH-алиасы: `~/.ssh/config`

Добавьте блоки в существующий файл (не дублируйте уже настроенные Host):

```sshconfig
Host idevconn
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_idevconn
    IdentitiesOnly yes
    AddKeysToAgent yes
    UseKeychain yes

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_itcampusdesc
    IdentitiesOnly yes
    AddKeysToAgent yes
    UseKeychain yes
```

`Host` — локальный псевдоним; `HostName` — настоящий сервер; `IdentityFile` — ключ для этого псевдонима. `UseKeychain` применим к macOS.

```bash
ssh -T git@idevconn
ssh -T git@github-work
```

GitHub сообщит имя аккаунта, для которого принят ключ. Для диагностики: `ssh -vT git@github-work`.

## 5. Центральный маршрутизатор: `~/.gitconfig`

**Добавьте** к существующей глобальной конфигурации:

```ini
[includeIf "gitdir:~/projects/work/"]
    path = ~/git-config/work.gitconfig

[includeIf "gitdir:~/projects/itcampus/"]
    path = ~/git-config/itcampus.gitconfig
```

Завершающий `/` означает, что правило распространяется на репозитории внутри директории. Условия применяются к расположению `.git`, а не к GitHub username или имени удалённого репозитория.

## 6. Настройки аккаунтов

`~/git-config/work.gitconfig`:

```ini
[user]
    name = Denis
    email = YOUR_PERSONAL_EMAIL

[core]
    sshCommand = ssh -i ~/.ssh/id_ed25519_idevconn -o IdentitiesOnly=yes
```

`~/git-config/itcampus.gitconfig`:

```ini
[user]
    name = Denis
    email = YOUR_WORK_EMAIL

[core]
    sshCommand = ssh -i ~/.ssh/github_itcampusdesc -o IdentitiesOnly=yes
```

Замените имена/email на настоящие значения. `core.sshCommand` выбирает ключ для Git-операций **внутри** соответствующего репозитория; локальные настройки и `GIT_SSH_COMMAND` могут его переопределить.

## 7. Clone: почему нужен SSH-алиас

![Clone routing](clone.png)

**При первом `git clone`** репозиторий ещё не создан, поэтому нельзя полагаться на условную конфигурацию его будущей директории. Используйте SSH-алиас прямо в URL:

```bash
cd ~/projects/work
git clone git@idevconn:iDEVconn/iCore.git

cd ~/projects/itcampus
git clone git@github-work:itcampuss/REPO.git
```

Замените `REPO` на реальное название. После клонирования можно пользоваться обычными `git pull` и `git push`.

## 8. Commit и Push: выбор происходит автоматически

![Push routing](push.png)

```bash
cd ~/projects/itcampus/REPO
git add .
git commit -m "Update project"
git push
```

Внутри существующего репозитория Git подключит `itcampus.gitconfig`: `user.name`/`user.email` определят автора коммита, а `core.sshCommand` задаст ключ для SSH. Push идёт в адрес, указанный в `origin`; права доступа по-прежнему проверяет GitHub.

## 9. Диагностика перед первым Push

```bash
git config --show-origin --get user.name
git config --show-origin --get user.email
git config --show-origin --get core.sshCommand
git remote -v
git config --show-origin --show-scope --list
```

Для проверки SSH-алиаса:

```bash
ssh -G github-work | grep -E '^(hostname|user|identityfile) '
ssh -T git@github-work
```

Если ключ неожиданно меняется, проверьте локальный `.git/config`, `GIT_SSH_COMMAND` и используемый remote. SSH-алиас помогает на этапе clone; внутри репозитория `core.sshCommand` может принудительно выбрать ключ даже для стандартного `github.com`.

## 10. Добавление третьего аккаунта

![Масштабирование](scale.png)

1. Создайте отдельный SSH-ключ и зарегистрируйте его публичную часть в третьем аккаунте.
2. Добавьте `Host client` в `~/.ssh/config`.
3. Создайте `~/git-config/client.gitconfig` с его `user` и `core.sshCommand`.
4. Добавьте в `~/.gitconfig`:

```ini
[includeIf "gitdir:~/projects/client/"]
    path = ~/git-config/client.gitconfig
```

5. Создайте `~/projects/client/` и клонируйте репозитории через `git@client:OWNER/REPO.git`.

**Результат:** несколько GitHub-аккаунтов, разные ключи и email, параллельная работа в IDE без ручного переключения. Это удобная маршрутизация Git, но не отдельные системные учётные записи.

### Для видеодемонстрации

Откройте два окна терминала, по одному репозиторию в каждом workspace. Покажите `git config --show-origin --get user.email`, `git config --show-origin --get core.sshCommand`, затем `ssh -T` для каждого алиаса. Завершите демонстрацию Push в два разных репозитория.

### Справочные материалы

- [Git conditional includes](https://git-scm.com/docs/git-config#_conditional_includes)
- [GitHub: connecting with SSH](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
