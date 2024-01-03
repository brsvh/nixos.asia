---
slug: zh-Hans/configuration-as-flake
---

# 将 `configuration.nix` 转换为 flake

官方安装程序生成的默认 NixOS `configuration.nix` 存在一个问题，即它不是“纯净”的，因此无法复制（参见[此处](https://www.tweag.io/blog/2020-07-31-nixos-flakes/#what-problems-are-we-trying-to-solve)），因为它仍然使用可变的 Nix 渠道（通常是[不推荐的](https://zero-to-nix.com/concepts/channels#the-problem-with-nix-channel)）。出于这个原因（以及其他原因），建议立即切换到使用 #[[flakes]] 作为我们的 NixOS 配置。这样做非常简单。只需在 `/etc/nixos` 中添加一个 `flake.nix` 文件：

```sh
sudo nvim /etc/nixos/flake.nix
```

添加以下内容：

```nix
# /etc/nixos/flake.nix
{
  inputs = {
    # 注意：将 "nixos-23.11" 替换为 configuration.nix 中的 system.stateVersion。如果您希望升级，也可以使用更高版本。
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11";
  };
  outputs = { self, nixpkgs }: {
    # 注意：安装程序设置的默认主机名为 'nixos'
    nixosConfigurations.nixos = nixpkgs.lib.nixosSystem {
      # 注意：如果您使用的是 ARM，请将其更改为 aarch64-linux
      system = "x86_64-linux";
      modules = [ ./configuration.nix ];
    };
  };
}
```

> [!note] 在上面的代码片段中，确保更改几个东西：
> - 将 `nixos-23.11` 替换为您的 `/etc/nixos/configuration.nix` 中的 [`system.stateVersion`](https://nixos.wiki/wiki/FAQ/When_do_I_update_stateVersion)。如果您希望立即升级，也可以使用更高版本，或使用 `nixos-unstable` 以获取最新版本。
> - 如果您使用的是 ARM，则 `x86_64-linux` 应该是 `aarch64-linux`

现在，`/etc/nixos` 技术上是一个 [[flakes|flakes]]。我们可以使用 `nix flake show` 命令“查看”这个 flake：

```sh
$ nix flake show /etc/nixos
error: 实验性 Nix 功能 'nix-command' 已禁用；使用 '--extra-experimental-features nix-command' 覆盖
```

哎呀，发生了什么？由于 flakes 是所谓的“实验性”功能，您必须手动启用它。我们现在将暂时启用它，稍后再永久启用。`--extra-experimental-features` 标志可用于启用实验性功能。让我们再试一次：

```sh
$ nix --extra-experimental-features 'nix-command flakes' flake show /etc/nixos
warning: 正在创建锁文件 '/etc/nixos/flake.lock'
error:
       … 在更新 flake 'path:/etc/nixos?lastModified=1702156351&narHash=sha256-km4AQoP/ha066o7tALAzk4tV0HEE%2BNyd9SD%2BkxcoJDY%3D' 的锁文件时

       error: 打开文件 '/etc/nixos/flake.lock' 时出错：权限被拒绝
```

有进展，但我们遇到了另一个错误 - Nix 不能写入 root 拥有的目录（它尝试创建 `flake.lock` 文件）。解决这个问题的一种方法是将整个配置移动到我们的家目录，这也为将其存储在 [[git]] 中做好了准备。我们将在下一节中执行此操作。

> [!info] `flake.lock` 
> Nix 命令会自

动生成（或更新）`flake.lock` 文件。这个文件包含 flake 输入的确切固定版本，这对于可复制性很重要。

{#homedir}
## 将配置移动到用户目录

将整个 `/etc/nixos` 目录移动到您的家目录，并获得对其的控制：

```sh
$ sudo mv /etc/nixos ~/nixos-config && sudo chown -R $USER ~/nixos-config
```

您的配置目录现在应该如下所示：

```sh
$ ls -l ~/nixos-config/
总计 12
-rw-r--r-- 1 srid root 4001 Dec  9 16:03 configuration.nix
-rw-r--r-- 1 srid root  224 Dec  9 16:12 flake.nix
-rw-r--r-- 1 srid root 1317 Dec  9 15:43 hardware-configuration.nix
```

现在让我们对其尝试 `nix flake show`，这次应该可以工作：

```sh
$ cd ~/nixos-config
$ nix --extra-experimental-features 'nix-command flakes' flake show
warning: 正在创建锁文件 '/home/srid/nixos-config/flake.lock'
path:/home/srid/nixos-config?lastModified=1702156518&narHash=sha256-nDtDyzk3fMfABicFuwqWitIkyUUw8BZ4SniPPyJNKjw%3D
└───nixosConfigurations
    └───nixos: NixOS 配置
```

瞧！顺便说一句，这个 flake 有一个输出，即 `nixosConfigurations.nixos`，它本身就是 NixOS 配置。

>[!info] 更多关于 Flakes 的信息
> 有关 flakes 的更多信息，请参阅 [[nix-rapid]]。

一旦 flake 化，我们可以使用相同的命令激活新配置，但我们必须额外传递 `--flake` 标志，即：

```sh
# '.' 是 flake 的路径，即当前目录。
$ sudo nixos-rebuild switch --flake .
```

如果一切顺利，您应该会看到类似这样的东西：

:::{.center}
![[nixos-rebuild-switch-flake.png]]
:::

太好了，现在我们有了一个纯净且可复制的 flake 化 NixOS 配置！
