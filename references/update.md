# 更新技能

默认更新源是 `https://gitee.com/ssocean/intuitive-deep-learning.git`。该仓库是技能代码和资源的唯一真相；安装目录不是用户工作区，不允许用户修改。用户询问“有没有更新”“检查更新”，或要求更新课程、同步最新版时，检查并直接应用更新，不要再次询问是否安装。

只保留程序生成的 `runtime_logs/` 和 `history/`。其他本地差异、残留旧文件、缓存和未跟踪文件都不保留。

## 判断是否为 Git 目录

将当前加载的 `SKILL.md` 所在目录记为 `skill_dir`，解析真实路径后检查 Git 顶层目录：

```bash
skill_dir="/absolute/path/to/intuitive-deep-learning"
skill_root="$(cd "$skill_dir" && pwd -P)"
repo_root="$(git -C "$skill_dir" rev-parse --show-toplevel 2>/dev/null || true)"
if [ -n "$repo_root" ]; then repo_root="$(cd "$repo_root" && pwd -P)"; fi
```

- `repo_root` 的真实路径等于 `skill_root`：技能目录是独立 Git 目录。
- `repo_root` 为空或指向技能目录的父目录：技能目录不是独立 Git 目录。父目录属于宿主项目，不要对宿主仓库执行任何更新命令。
- 技能目录内存在 `.git`，但 `git rev-parse` 失败：Git 元数据已损坏，按非 Git 目录完整替换，不要尝试修复或复用损坏的元数据。

## 不是独立 Git 目录

把 Gitee 默认分支浅克隆到技能目录的同一父目录并验证关键文件。用临时仓库的索引比较整个安装目录；相同时报告已经是最新版，不改动目录。存在任何非忽略差异时，停止服务，保留允许的运行状态，然后用下载好的目录整体替换安装目录。整体替换后安装目录将成为独立 Git 仓库。

```bash
parent_dir="$(dirname "$skill_root")"
tmp_dir="$(mktemp -d "$parent_dir/.intuitive-deep-learning-update.XXXXXX")"
git clone --depth 1 https://gitee.com/ssocean/intuitive-deep-learning.git "$tmp_dir/latest"
test -f "$tmp_dir/latest/SKILL.md"
test -f "$tmp_dir/latest/modules/index.json"
test -f "$tmp_dir/latest/scripts/run-lesson-page.sh"
new_version="$(git -C "$tmp_dir/latest" rev-parse --short HEAD)"
differences="$(git --git-dir="$tmp_dir/latest/.git" --work-tree="$skill_root" status --porcelain --untracked-files=all)"
```

仅当 `differences` 非空时执行：

```bash
bash "$skill_root/scripts/start-all-services.sh" --stop
for path in settings.json runtime_logs history; do
  if [ -e "$skill_root/$path" ]; then cp -a "$skill_root/$path" "$tmp_dir/latest/"; fi
done
backup_dir="$tmp_dir/previous"
mv "$skill_root" "$backup_dir"
if mv "$tmp_dir/latest" "$skill_root"; then
  rm -rf "$backup_dir"
else
  mv "$backup_dir" "$skill_root"
  exit 1
fi
git -C "$skill_root" rev-parse --is-inside-work-tree
```

克隆、校验或服务停止失败时，不要移动现有目录。替换失败时立即恢复 `backup_dir`。完成或确认无需更新后删除临时目录。

## 是独立 Git 目录

读取 `origin`，只接受 `https://gitee.com/ssocean/intuitive-deep-learning.git`、省略 `.git` 的同地址或对应的 Gitee SSH 地址。远端缺失或不匹配时停止，不要从未知仓库更新，也不要擅自修改远端。

获取 Gitee 当前默认分支，不要写死 `main` 或 `master`。比较远端提交、本地 `HEAD` 和工作区状态；三者一致时报告已经是最新版。只要提交或文件状态不同，就把所有受控文件强制对齐远端，并删除除 Git 忽略项以外的未跟踪文件。不要保留本地代码修改、本地提交或分叉历史。

```bash
origin_url="$(git -C "$skill_root" remote get-url origin)"
git -C "$skill_root" fetch --prune origin
git -C "$skill_root" remote set-head origin --auto
target_ref="$(git -C "$skill_root" symbolic-ref refs/remotes/origin/HEAD)"
old_version="$(git -C "$skill_root" rev-parse --short HEAD)"
new_version="$(git -C "$skill_root" rev-parse --short "$target_ref")"
differences="$(git -C "$skill_root" status --porcelain --untracked-files=all)"
```

仅当 `old_version` 与 `new_version` 不同或 `differences` 非空时执行：

```bash
bash "$skill_root/scripts/start-all-services.sh" --stop
git -C "$skill_root" reset --hard "$target_ref"
git -C "$skill_root" clean -fd
```

`git clean -fd` 不删除 `.gitignore` 已忽略的 `settings.json`、`runtime_logs/` 和 `history/`。重置或清理失败时报告更新失败，不要声称已经完成。

## 完成与失败

安装目录不可写、网络受限或需要额外权限时，按宿主环境的审批机制申请访问 Gitee 和写入技能目录。不要把权限、下载或获取失败误报为已经是最新版。

只有比较结果一致或目录已成功对齐远端后才能报告完成。更新后重新读取新的 `SKILL.md`、本文件和 `modules/index.json`，向用户报告结果及 `new_version`；发生更新时同时报告 `old_version`（非 Git 目录可能没有该值）。用户还要求打开课程时，再按新说明重新启动课程。
