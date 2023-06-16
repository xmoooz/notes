# setup a minimal OS for development

I choose Archlinux, nixos is also good, but the main reason for choosing arch instead of nix is its  community and support.
After downloading virtualbox app and archlinux iso, define a simple vm by these specs:
cpu: 4core, ram: 8GB, hdd: 100GB vdi(dynamic allocated)
iso: archlinux-2023.06.01-x86_64.iso (download the updated version from offical site)
after you create a vm can change its setting like:
vga: vmsvga, no shared clipboard, no drag & drop, no audio, floppy disabled in boot order
after all above, hit start button, now follow the below steps:

select Arch linux Install medium option & hit enter, then:
1. Check internet connectivity: `ping 8.8.8.8`
2. Check ntp status: `timedatectl status`
3. Setup ntp: `timedatectl set-ntp true`
4. Check ntp status again: `timedatectl status`
5. List hdd: `fdisk -l`
6. Make partitions: `cfdisk`, then select: `dos` option
7. Create partition: 50gb, 50gb (this is based on your prefrence, this is good for me)
   both of them are primary and their type is linux(83), first partition is bootable,
   then select: write and type: yes and then select: quit
8. Format disks:
   `mkfs.ext4 /dev/sda1`
   `mkfs.ext4 /dev/sda2`
9. Mount disks:
   `mount /dev/sda1 /mnt`
   `mkdir /mnt/home`
   `mount /dev/sda2 /mnt/home`
10. Final check hdd:
    `parted`, followed by `print` command and `quit` command too, or use `cfdisk`
11. Install arch: `pacstrap /mnt base linux` (I don't use pacstrap -K command)
12. `genfstab -U /mnt >> /mnt/etc/fstab`
13. Check result of above command: `cat /mnt/etc/fstab`
14. Change root: `arch-chroot /mnt`
15. Config lang:
    first we need a text editor, I choose the nano:
    `pacman -Syy`
    `pacman -Sy archlinux-keyring`
    `pacman -S --needed nano`
    Uncomment lang you want from this file, I uncomment this: en_US.UTF-8 UTF-8 and save it.
    `nano /etc/locale.gen`
    `locale-gen`
    `echo LANG=en_US.UTF-8 > /etc/locale.conf`
16. Config clock:
    `ln -fs /usr/share/zoneinfo/Europe/London /etc/localtime`
    `hwclock --systohc --utc`
17. Config hostname:
    `echo dev > /etc/hostname` (replace 'dev' with your own)
18. Config grub:
    `pacman -S --needed grub`
    `grub-install /dev/sda`
    `grub-mkconfig -o /boot/grub/grub.cfg`
19. Config dhcpcd:
    `pacman -S --needed dhcpcd`
    `systemctl enable dhcpcd`
20. set root password:
    `passwd` and then type your password: 1234 (replace '1234' with your own)
21. Create user:
    `useradd -m -g users -G wheel -s /bin/bash archi` (replace 'archi' with your own)
22. Add password to the user:
    `passwd archi` and then type your password: 12345 (replace '12345' with your own)
23. install 2 usefull pkg:
    `pacman -S --needed polkit sudo`
    `EDITOR=nano visudo`, then uncomment line : to allow wheel group allow execute any command
24. And finally:
    `exit`
    `shutdown -h now`
25. Remove iso file from vm storage setting,
    now because we have installed all necessary pkgs, it's a good time to take a full clone of vm.
26. after taking a full backup of vm, it's good time to install more pkg and customize it, with
    having this rule: minimum is god.
27. Install general vga: `sudo pacman -S --needed xf86-video-vesa`
28. Install xorg: `sudo pacman -S --needed xorg-server xorg-xinit xorg-xrandr`
29. Install: `sudo pacman -S --needed rxvt-unicode dmenu ttf-fira-code`
30. Install i3: `sudo pacman -S --needed i3-gaps i3status`
    run this for updating fonts cache: `fc-cache -fv`
    Config i3:
    `sudo nano /etc/X11/xinit/xinitrc`
    add this to EOF: `exec i3`
    and comment other exec command with adding `#` in begaining of line, 
    run i3 with this command: `startx` and for stop i3 with: `i3 exit`.
    because we install i3 window manager, so all other commands run inside i3.
31. install fish shell and set it as default shell:
    `sudo pacman -S --needed fish`
    `chsh -s /usr/bin/fish`
32. Active clipboard sharing so we can copy & paste from os to vm:
    first in vm setting allow to have shared clipboard bidirectional, then:
    `sudo pacman -S --needed hwinfo virtualbox-guest-utils`
    `systemctl start vboxservice.service`, 
    `VBoxClient --clipboard`,
    now you can copy & paste texts from/inside vm.
    after restarting the vm, only run `VBoxClient --clipboard` to enable shared clipboard.
    you can make scripts to automate all things ;)
33. config time:
    lets config fish:(define some alias)
    nano ~/.config/fish/config.fish:
    set fish_greeting
    alias hdo="shutdown -h now"
    alias rdo="shutdown -r now"
    alias ll="ls -la"
    lets config terminal:
    create .Xresources file in home dir and add these lines to file:
    nano ~/.Xresources
    URxvt.font:xft:Fira Code:size=18
    ! Dracula Xresources palette
    *.foreground: #F8F8F2
    *.background: #282A36
    *.color0:     #000000
    *.color1:     #FF5555
    *.color2:     #50FA7B
    *.color3:     #F1FA8C
    *.color4:     #BD93F9
    *.color5:     #FF79C6
    *.color6:     #8BE9FD
    *.color7:     #BFBFBF
    *.color8:     #4D4D4D
    *.color9:     #FF6E67
    *.color10:    #5AF78E
    *.color11:    #F4F99D
    *.color12:    #CAA9FA
    *.color13:    #FF92D0
    *.color14:    #9AEDFE
    *.color15:    #E6E6E6
    and after saving it use this command:
    `xrdb ~/.Xresources`
    now if you open new terminal you will new theme and fonts.
    lets config i3:
    nano ~/.config/i3/config:
    bar {
            status_command i3status
            font pango:Fira 12
    }
    `exec_always VBoxClient --vmsvga`
    lets config display:
    change gui in virtualbox setting to :vmsvga and zoom to 125%
    then if the result is not what you want(it depends on screedn size), you can change them and also
    can change reolution by View menu bar and these options: 
    `adjust window size` or `auto-resize guest display`
34. Now I make second backup of the vm.
35. install some pks: `sudo pacman -S --needed zip unzip unrar wget curl tree`
36. install some pks: `sudo pacman -S --needed git openssh bat base-devel`
37. install yay: https://github.com/Jguer/yay
38. install google chrome by yay: `yay -S --needed google-chrome`
39. install vscode by yay: `yay -S --needed visual-studio-code-bin`
40. install fisher(fish plugin manager): https://github.com/jorgebucaran/fisher
41. install asdf: https://github.com/asdf-vm/asdf
    also use this command too in part3 of installation guide: `yay -S --needed asdf-vm`
42. install docker & docker-compose: 
    `sudo pacman -S --needed docker docker-compose`
    `systemctl enable docker.service`
    this link is udefull:
    https://docs.docker.com/engine/install/linux-postinstall/
43. now it is ready for starting developemnt,
    can download any version of any language by asdf,
    can setup any env in vscode by profiles,
    can download any docker imgae
    ...
    

