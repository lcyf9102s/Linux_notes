# Linux-memo
Arch Linux learning notes ( Arch Wiki: https://archlinux.org/ )

1. okular pdf注解：F6 https://docs.kde.org/stable5/en/okular/okular/annotations.html

2. GitHub markdown：https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax  
   markdown换行：末尾打两个空格

3. Update grub after installing custom kernel:
   ```
   $ sudo grub-mkconfig -o /boot/grub/grub.cfg
   ```

4. C++中矩阵要初始化。如定义一个矩阵：
   ```c++
   double data[row][column];
   ```
   不初始化会出现Bus error。初始化，给每个元素赋一个初始值：
   ```c++
   for(int i=0;i<=row-1;i++)
    {
      for(int j=0;j<=column-1;j++)
      {
        data[i][j]=0;
      }
    }
   ```

5. linux上编译Geant4（已在opensuse、wsl2、Arch Linux上测试）:
   ```
   $ cmake -DCMAKE_INSTALL_PREFIX=/path/to/install \
      -DGEANT4_USE_OPENGL_X11=ON \
      -DGEANT4_USE_RAYTRACER_X11=ON \
      -DGEANT4_INSTALL_DATA=ON \
      -DGEANT4_INSTALL_DATADIR=/path/to/datasets \
      -DGEANT4_USE_QT=ON \
      -DGEANT4_BUILD_MULTITHREADED=ON \
      /path/to/src
   ```

6. Opensuse tumbleweed下nvidia导致休眠恢复失败：在`/etc/default/grub`中添加内核启动参数`nouveau.blacklist=1`

7. 高分屏再x11下显示字体图标过小（如surface pro 7）：kde下，在设置中将缩放改为200%，然后在~/.xprofile中

8. pacman更新时出现key相关的问题（如无法导入、corrupted）：删除/etc/pacman.d/gnupg,再pacman-key --init
   ```
   # rm -rf /etc/pacman.d/gnupg
   # pacman-key --init
   ```

9. git本地仓库push到远程：
   ```
   git config --global user.name "yourname"
   git config --global user.email example@example.com
   git add <filename>
   git commit -m "***"
   git push origin main
   ```
   本地与远程同步：
   ```
   git pull
   ```

10. git使用ssh进行clone和push，publickey添加到GitHub：
    ```
    ssh-keygen -t rsa -C "your_email@example.com"
    git clone git@github.com:lcyf9102s/ArchLinux-Learning.git
    ```

11. 给linux kernel source手动打patch：
    ```
    patch -p2 < name.patch
    ```
    P.s. 要输入对应需要打patch的文件的相对或绝对路径
12. wayland下Geant4可视化窗口无显示问题：
    运行前添加环境变量：`XDG_SESSION_TYPE=x11`
    ```
    $ export XDG_SESSION_TYPE=x11
    ```
13. VSCode factional scaling blurring issue fix for HiDPI Display (wayland):
    ```
    code --enable-features=UseOzonePlatform,WaylandWindowDecorations --ozone-platform=wayland
    ```
14. Arch Linux 使用systemd-boot引导时开启secure boot (surface pro)：https://github.com/linux-surface/linux-surface/wiki/Secure-Boot

15. 使用Mosh，以及private key登录，指定ssh端口：
    ```
    mosh --ssh="ssh -p PORT_NUMBER -o "IdentitiesOnly=yes" -i /path/to/private/key" user@ip
    ```
    scp:
    ```
    scp -P PORT_NUMBER -i /path/to/private/key FILENAME user@ip:/target/path/
    ```
16. 批量修改扩展名：
    ```
    find `pwd` -name '*.patch' | awk -F '.' '{print $1}' | xargs -I {} mv {}.patch {}.mypatch
    ```   
17. Linux修复dirty ntfs mount：
    ```
    sudo ntfsfix -d /dev/sdb1
    sudo mount -t ntfs3 /dev/sdb1 /mnt/Data
    ```
18. Windows下ctex解决编译缺少revtex4-1.cls：
    ```
    (1) 下载revtex4-1.zip，解压得到revtex4-1-tds.zip，再解压找到tex/latex/revtex，重命名文件夹为revtex4-1，放进 `CTEX安装目录/MiKTeX/tex/latex/`；
    (2) 打开winedt，找到TeX - MiKTeX Console，切换至管理员模式，再点击任务 - 刷新文件名数据库。
    ```
19. Fix tensorflow NUMA node zero:
    ```
    # echo 0 | sudo tee -a /sys/bus/pci/devices/0000\:01\:00.0/numa_node
    $ cat /sys/bus/pci/devices/0000\:01\:00.0/numa_node
    ```
20. Changing Ollama model path:
    ```
    $ OLLAMA_MODELS=/path/to/model ollama serve
    ```
21. Passing current environment variables to sudo in Linux terminal:
    ```
    sudo -E (your command)
    ```
22. Local git recover deleted files:
    ```
    git fetch --all
    git reset --hard origin/main
    git pull
    ```
23. Check image ppi:
    ```
    identify -format "%[resolution.x]x%[resolution.y] %[units]" image.png
    ```
24. Build caddy with cloudflare-ip support using xcaddy:
    ```
    go/bin/xcaddy build --with github.com/WeidiDeng/caddy-cloudflare-ip
    ```
25. Building Mesa on raspberry pi (enable deb-src repo):
    ```
    sudo apt build-dep mesa
    (then follow the instructions on mesa git repo)
    ```
26. Raspberry Pi OS start GUI from CLI:
    ```
    $ labwc
    ```
27. Disable/enable Android's automatic captive portal detection on Google Pixels:
    ```
    $ adb shell settings put global captive_portal_mode 0/1
    ```
