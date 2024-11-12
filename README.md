# PPPwn c++ for Android

This is the C++ rewrite of [PPPwn](https://github.com/TheOfficialFloW/PPPwn) by [xfangfang](https://github.com/xfangfang/PPPwn_cpp) modified to compile for Android

# Features

- The same as xfanfang one
- Can compile from NDK or Termux app
- Supported target Android versions from KitKat 4.4 up

# Building

### Building from native Linux machine

Download your desired NDK version through Android Studio or directly from official [github](https://github.com/android/ndk/releases). If you want to compile for Android 4.4.x targets you must download version [r25c here](https://dl.google.com/android/repository/android-ndk-r25c-linux.zip), otherwise you can use a more recent one.

Extract the files in your <home>/Android/Sdk/ndk or other preferred folder.

Clone this repository with
```shell
git clone --recursive https://github.com/deviato/PPPwn_cpp_android.git
cd PPPwn_cpp_android
```
Edit `CMakeLists.txt` file and change the options to reflect your system and desired target (info inside the file).

Run these commands to build the binary:
```shell
cmake -B build
cmake --build build -t pppwn
```
The resulting binary will be in build/pppwn

### Building from Termux app directly on your phone

Download Termux app (not GooglePlay but [F-Droid version here](https://f-droid.org/repo/com.termux_118.apk)) and install on your phone.
Open the app and run these commands:
```shell
apt -y update && apt -y upgrade && apt -y update
# (respond y to every question)
apt install -y git build-essential binutils file ndk-multilib-native-static termux-elf-cleaner tsu
# Clone this repository
git clone --recursive https://github.com/deviato/PPPwn_cpp_android.git
cd PPPwn_cpp_android
# Build for your device
cmake -B build -DTERMUX=1
cmake --build build -t pppwn
# If no errors, your target is in build/pppwn, but you need to realign TLS before running:
termux-elf-cleaner build/pppwn
# Optionally strip debug symbols with:
strip build/pppwn
# You can now try to run the build with root:
tsu
build/pppwn
```

# Usage

### show help

```shell
pppwn
```

### list interfaces

```shell
pppwn list
```

### run the exploit

```shell
pppwn --interface en0 --fw 1100 --stage1 "stage1.bin" --stage2 "stage2.bin" --timeout 10 --auto-retry
```

- `-i` `--interface`: the network interface which connected to ps4
- `--fw`: the firmware version of the target ps4 (default: `1100`)
- `-s1` `--stage1`: the path to the stage1 payload (default: `stage1/stage1.bin`)
- `-s2` `--stage2`: the path to the stage2 payload (default: `stage2/stage2.bin`)
- `-t` `--timeout`: the timeout in seconds for ps4 response, 0 means always wait (default: `0`)
- `-wap` `--wait-after-pin`: the waiting time in seconds after first round CPU pinning (default: `1`)
- `-gd` `--groom-delay`: wait for 1ms every `groom-delay` rounds during Heap grooming (default: `4`)
- `-bs` `--buffer-size`: PCAP buffer size in bytes, less than 100 indicates default value (usually 2MB) (default: `0`)
- `-a` `--auto-retry`: automatically retry when fails or timeout
- `-nw` `--no-wait-padi`: don't wait one more [PADI](https://en.wikipedia.org/wiki/Point-to-Point_Protocol_over_Ethernet#Client_to_server:_Initiation_(PADI)) before starting the exploit
- `-rs` `--real-sleep`: use CPU for more precise sleep time (Only used when execution speed is too slow)
- `-old` `--old-ipv6`: use previous IPv6 address to exploit (Only used when the exploit fails)
- `--web`: use the web interface
- `--url`: the url of the web interface (default: `0.0.0.0:7796`)

Supplement:

1. For `--timeout`, waiting for `PADI` is not included, which allows you to start `pppwn_cpp` before the ps4 is launched.
2. For `--no-wait-padi`, by default, `pppwn_cpp` will wait for two `PADI` request, according to [TheOfficialFloW/PPPwn/pull/48](https://github.com/TheOfficialFloW/PPPwn/pull/48) this helps to improve stability. You can turn off this feature with this parameter if you don't need it.
3. For `--wait-after-pin`, according to [SiSTR0/PPPwn/pull/1](https://github.com/SiSTR0/PPPwn/pull/1) set this parameter to `20` helps to improve stability (not work for me), this option not used in web interface.
4. For `--groom-delay`, This is an empirical value. The Python version of pppwn does not set any wait at Heap grooming, but if the C++ version does not add some wait, there is a probability of kernel panic on my ps4. You can set any value within 1-4097 (4097 is equivalent to not doing any wait).
5. For `--buffer-size`, When running on low-end devices, this value can be set to reduce memory usage. I tested that setting it to 10240 can run normally, and the memory usage is about 3MB. (Note: A value that is too small may cause some packets to not be captured properly)

# Credits

Big thanks to xfangfang's magical work, you are my hero.
And to FloW's magical work, which is xfangfang's hero (and consequently mine too).
