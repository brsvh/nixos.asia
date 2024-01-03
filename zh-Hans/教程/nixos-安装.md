# 使用 Git 创建 Flakes 配置来安装 NixOS

本教程将指导您完成安装 #[[nixos]] 的必要步骤，同时启用 [[flakes]] 并在 [[git]] 仓库中跟踪所得到的系统配置。

>[!info] 欢迎来到 [[nixos]] 教程系列
> 本页是旨在帮助 Linux/macOS 用户轻松使用 [[nixos]] 作为他们主要操作系统的一系列教程中的第一篇。

{#install}
## 安装 NixOS

- 从[此处](https://nixos.org/download#download-nixos)下载最新的 NixOS ISO 映像。选择适合您的 CPU 架构的 GNOME（或 Plasma）图形 ISO 映像。
- 创建一个可启动的 USB 闪存驱动器（[此处有说明](https://nixos.org/manual/nixos/stable/index.html#sec-booting-from-usb)），并从中启动计算机。

NixOS 将启动进入一个带有安装程序的图形环境。

:::{.center}
![[nixos-installer.png]]
:::

按照安装向导操作；它与其他发行版非常相似。安装完成后，重新启动进入您的新系统。您将看到登录界面。

- 用您在安装过程中创建的用户登录，并输入您设置的密码。
- 然后从“活动”菜单中打开“控制台”应用。

{#edit}
## 您的第一个 `configuration.nix` 更改

系统配置包括从分区布局到内核版本、软件包和服务等所有内容。它定义在 `/etc/nixos/configuration.nix` 中。`/etc/nixos` 目录看起来像这样：

```sh
$ ls -l /etc/nixos
-rw-r--r-- 1 root root 4001 Dec  9 16:03 configuration.nix
-rw-r--r-- 1 root root 1317 Dec  9 15:43 hardware-configuration.nix
```

>[!info] 什么是 `hardware-configuration.nix`？
> 硬件特定配置（例如：要挂载的磁盘分区）定义在 `/etc/nixos/hardware-configuration.nix` 中，该文件被 `configuration.nix` 作为 [[modules|module]] `import`。

所有系统更改都需要对此 `configuration.nix` 进行更改。例如，为了“安装”或“卸载”一个软件包，我们需要编辑这个 `configuration.nix` 并激活它。现在就让我们这样做，以安装 [neovim](https://neovim.io/) 文本编辑器。NixOS 默认包括 nano 编辑器：

```sh
sudo nano /etc/nixos/configuration.nix
```

>[!tip] Nix 语言
> 这些 `*.nix` 文件是用 [[nix]] 语言编写的。

在文本编辑器中进行以下更改：

- 在 `environment.systemPackages` 下添加 `neovim`
- [可选] 取消注释 `services.openssh.enable = true;` 以启用 SSH 服务器

按 <kbd>Ctrl+X</kbd> 退出 nano。

您的 `configuration.nix` 现在应该看起来像这样：

```nix
# /etc/nixos/configuration.nix
{
  ...
  environment.systemPackages = with pkgs; [
    neovim
  ];
  ...
  services.openssh.enable = true;
  ...
}
```

一旦将 `configuration.nix` 文件保存到磁盘，您必须使用 [nixos-rebuild](https://nixos.wiki/wiki/Nixos-rebuild) 命令激活新配置：

```sh
sudo nixos-rebuild switch
```

这需要几分钟时间来完成 - 因为它需要从官方 [[cache|binary cache]]（`cache.nixos.org`）中获取 neovim 及其依赖项。完成后，您应该会
