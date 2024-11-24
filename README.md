# Apple Music ALAC / Dolby Atmos Downloader

Original script by Sorrow. Modified by [@zhaarey](https://github.com/zhaarey) and me to include some fixes and improvements.

> [!IMPORTANT]
> Install MP4Box before continuing. Ensure it's correctly added to the environment variables.\
> You can do this by entering `apt install wget git golang gpac ffmpeg nano -y` in your terminal.

## New Features

- MP4Box will be called automatically to package EC3 as M4A file.
- Changed the directory structure to `ArtistName\AlbumName`; For Atmos files, its directory structure has changed to `ArtistName\AlbumName [Atmos]` and files are moved into `AM-DL-Atmos downloads`.
- Added support for overall completion status display upon finishing the run.
- Automatically embed cover art and LRC lyrics (requires `media-user-token`).
- Supports `check` with `main` which can take a text address or API database.
- Added `get-m3u8-from-device` (set to `true`) and set port `adb forward tcp:20020 tcp:20020` to get M3U8 from emulator.
- Added support templates for folder and file.
- Added support for downloading all albums from an artist: `go run main.go https://music.apple.com/us/artist/taylor-swift/159260351 --all-album`
- Added wrapper mode (currently only runs on Linux); decryption speed is extremely fast.
- Added support for length limitation with `limit-max` (default value of `limit-max` is 200).
- Added support for synchronized and unsynchronized lyrics.

### Supported Formats

- ALAC (audio-alac-stereo)
- EC3 (audio-atmos / audio-ec3)

For AAC downloads, it is recommended to use [WorldObservationLog's AppleMusicDecrypt](https://github.com/WorldObservationLog/AppleMusicDecrypt).

### Supported formats for AppleMusicDecrypt

- ALAC (audio-alac-stereo)
- EC3 (audio-atmos / audio-ec3)
- AC3 (audio-ac3)
- AAC (audio-stereo)
- AAC-binaural (audio-stereo-binaural)
- AAC-downmix (audio-stereo-downmix)

## How To Use

**1. Start up the decoding daemon:**

- With **frida server**:
    1. Create a virtual device on Android Studio with an image that doesn't have Google APIs.
    2. Install this version of Apple Music: [Apple Music 3.6.0 beta](https://www.apkmirror.com/apk/apple/apple-music/apple-music-3-6-0-beta-release/apple-music-3-6-0-beta-4-android-apk-download/). You will also need SAI to install it: [SAI on F-Droid](https://f-droid.org/pt_BR/packages/com.aefyr.sai.fdroid/).
    3. Launch Apple Music and sign in to your account (subscription required).
    4. Port forward 10020 TCP:
    ```sh
    adb forward tcp:10020 tcp:10020
    ```
    6. Start the **frida** agent with the command bellow:
    ```sh
    frida -U -l agent.js -f com.apple.android.music
    ```
- With **wrapper**:
1. Run the following command to download **wrapper**:
```sh
wget "https://github.com/itouakirai/wrapper/releases/download/linux/wrapper.linux.x86_64.tar.gz" && mkdir wrapper && tar -xzf wrapper.linux.x86_64.tar.gz -C wrapper
```
2. Start the wrapper daemon:
- With command:
    1. cd to wrapper directory
    ```sh
    cd wrapper
    ```
    \
    2. Start the daemon with following command:
    ```sh
    ./wrapper -D 10020 -M 20020 -L username:password
    ```
    \
    Replace both `username` and `password` with your Apple Music account credentials.
    \
    \
    3. Once the service is running on background. move on to Step 2 by opening another terminal.

> [!WARNING]
> The follow script is still in testing stage, I cannot guarantee the script 100% works.
- With **python script** (beta):
    1. Copy the **python script** below:
    ```python
    #!/usr/bin/env python3
    import os
    import sys
    import time
    import subprocess
    import threading
    import getpass
    from datetime import datetime
    import multiprocessing

    def get_credentials():
    username = input("Enter username (add 86 prefix for Chinese mainland): ")
    password = getpass.getpass("Enter password: ")
    return f"{username}:{password}"

    def log_output(timestamp, message):
    with open("wrapper_log.txt", "a") as log:
        log.write(f"[{timestamp}] {message}\n")

    def handle_2fa():
    code = input("Enter 2FA code: ")
    return code + "\n"

    def background_process(log_file):
    # Redirect standard file descriptors
    with open(os.devnull, 'r') as devnull:
        os.dup2(devnull.fileno(), sys.stdin.fileno())
    with open(log_file, 'a') as log:
        os.dup2(log.fileno(), sys.stdout.fileno())
        os.dup2(log.fileno(), sys.stderr.fileno())

    def read_output(process):
    m3u8_seen = False
    while True:
        line = process.stdout.readline()
        if not line and process.poll() is not None:
            break
        if line:
            timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            line = line.decode('utf-8', errors='replace').strip()
            
            # Print to console only before backgrounding
            if not m3u8_seen:
                print(f"[{timestamp}] {line}")
            
            log_output(timestamp, line)
            
            # Check for login failure
            if "login failed" in line.lower():
                print("\nLogin failed. Exiting...")
                sys.stdout.flush()
                time.sleep(0.1)  # Give time for the message to be printed
                process.terminate()
                sys.exit(1)
            
            if "listening m3u8 request on" in line and not m3u8_seen:
                m3u8_seen = True
                print("\nService is ready. Moving to background...")
                sys.stdout.flush()
                
                # Create new background process using multiprocessing
                ctx = multiprocessing.get_context('spawn')
                p = ctx.Process(target=background_process, args=("wrapper_log.txt",))
                p.daemon = True
                p.start()
                
                # Exit the parent process
                os._exit(0)
                    
            if "2FA code:" in line or "verification code" in line:
                code = handle_2fa()
                process.stdin.write(code.encode())
                process.stdin.flush()

    def main():
    try:
        # Change to wrapper directory
        os.chdir("wrapper")
        
        # Get credentials
        credentials = get_credentials()
        
        # Start wrapper process
        process = subprocess.Popen(
            ["./wrapper", "-D", "10020", "-M", "20020", "-L", credentials],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            bufsize=0,
            universal_newlines=False
        )

        # Start output reading thread
        output_thread = threading.Thread(target=read_output, args=(process,))
        output_thread.daemon = True
        output_thread.start()

        # Wait for process to complete or error
        process.wait()
        
        # If process exits with non-zero status, exit the script
        if process.returncode != 0:
            sys.exit(process.returncode)

    except KeyboardInterrupt:
        print("\nInterrupted by user")
        if 'process' in locals() and process.poll() is None:
            process.terminate()
            process.wait(timeout=5)
        sys.exit(1)
    except Exception as e:
        print(f"Error: {str(e)}")
        sys.exit(1)

    if __name__ == "__main__":
    multiprocessing.set_start_method('spawn')
    main()
    ```
    \
    and store it to your root user directory.
    \
    2. Run the code and enter your Apple Music credentials.
    \
    3. Once the service is being moved to background. move on to Step 2 in the same terminal. 

**2. Run the `main.go` file with your preferred options:**
    \
    e.g. To download the whole album: 
    ```sh
    go run main.go https://music.apple.com/us/album/whenever-you-need-somebody-2022-remaster/1624945511
    ```
    \
    To download some selected songs in the album: 
    ```sh
    go run main.go --select https://music.apple.com/us/album/whenever-you-need-somebody-2022-remaster/1624945511
    ``` 
    \
    Once prompted, enter your numbers separated by spaces.
    
    To download the entire playlist: 
    ```sh
    go run main.go https://music.apple.com/us/playlist/taylor-swift-essentials/pl.3950454ced8c45a3b0cc693c2a7db97b
    ``` 
    or 
    ```sh
    go run main.go https://music.apple.com/us/playlist/hi-res-lossless-24-bit-192khz/pl.u-MDAWvpjt38370N
    ```
    \
    To download all albums from an artist:
    ```sh
    go run main.go https://music.apple.com/us/artist/taylor-swift/159260351 --all-album
    ```
    \
    To download songs with Dolby Atmos® support: 
    ```sh
    go run main.go --atmos https://music.apple.com/us/album/1989-taylors-version-deluxe/1713845538
    ```
    \
    \
    and replace the URL above with your vaild URL

### For Downloading Lyrics

1. Open Apple Music and log in.
2. Open the Developer tools, Click `Application -> Storage -> Cookies -> https://music.apple.com`.
3. Find the cookie named `media-user-token` and copy its value.
4. Paste the cookie value obtained in step 3 into the `config.yaml` and save it.
5. Start the script as usual.

---

**Note:** For detailed instructions in Chinese, see [method three in the original documentation](https://telegra.ph/Apple-Music-Wrapper-On-WSL1-07-21).
