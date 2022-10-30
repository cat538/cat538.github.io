## Linux 管理多版本工具链

sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 10 \
    --slave /usr/bin/g++ g++ /usr/bin/g++-9 \

sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 20 \
    --slave /usr/bin/g++ g++ /usr/bin/g++-12 \
