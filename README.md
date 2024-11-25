# Apple Music ALAC / Dolby Atmos Downloader

Original script by Sorrow. Credit to [@zhaarey](https://github.com/zhaarey) and modified by me to include some fixes and improvements.

> [!IMPORTANT]
> Install MP4Box before continuing. Ensure it's correctly added to the environment variables.\
> You can do this by entering `apt install wget git golang gpac ffmpeg nano -y` in your terminal.

## New Features

- MP4Box will be called automatically to package EC3 as M4A file.
- Changed the directory structure to `ArtistName\AlbumName`; For Atmos files, its directory structure has changed to `ArtistName\AlbumName [Atmos]` and files are moved into `AM-DL-Atmos downloads`.
- Add support for overall completion status display upon finishing the run.
- Automatically embed cover art and LRC lyrics (requires `media-user-token`).
- Supports `check` with `main`, which can take a text address or API database.
- Add `get-m3u8-from-device` (set to `true`) and set port `adb forward tcp:20020 tcp:20020` to get M3U8 from the emulator.
- Add support templates for folders and files.
- Add support for downloading all albums from an artist: `go run main.go https://music.apple.com/us/artist/taylor-swift/159260351 --all-album`
- Add wrapper integration with its insane decryption speed rate (Supports Linux OS currently)
- Add support for length limitation with `limit-max` (default value of `limit-max` is 200).
- Add support for synchronized and unsynchronized lyrics.
- Add support for decoding files for arm64 platforms.

### Supported Formats

- ALAC `(audio-alac-stereo)`
- EC3 `(audio-atmos / audio-ec3)`

For AAC downloads, it is recommended to use [WorldObservationLog's AppleMusicDecrypt](https://github.com/WorldObservationLog/AppleMusicDecrypt).

### Supported formats for AppleMusicDecrypt

- ALAC `(audio-alac-stereo)`
- EC3 `(audio-atmos / audio-ec3)`
- AC3 `(audio-ac3)`
- AAC `(audio-stereo)`
- AAC-binaural `(audio-stereo-binaural)`
- AAC-downmix `(audio-stereo-downmix)`

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
        - For x86 platforms:
        ```sh
        frida -U -l agent.js -f com.apple.android.music
        ```
        - For arm64 platforms:
       ```sh
        frida -U -l agent-arm64.js -f com.apple.android.music
       ```
- With **wrapper**:
1. Run the following command to download **wrapper**:
```sh
wget "https://github.com/itouakirai/wrapper/releases/download/linux/wrapper.linux.x86_64.tar.gz" && mkdir wrapper && tar -xzf wrapper.linux.x86_64.tar.gz -C wrapper
```
2. Start the **wrapper daemon**:
- With command:
    1. `cd` to wrapper directory
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
> The following script is still in the testing stage; I do not guarantee the script will fully work.
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
    import signal

    def get_instance_info():
    try:
        # Get detailed process information
        result = subprocess.run(
            ['ps', 'aux'], 
            stdout=subprocess.PIPE, 
            stderr=subprocess.DEVNULL,
            text=True
        )
        
        # Filter wrapper processes
        wrapper_processes = []
        for line in result.stdout.splitlines():
            if './wrapper -D' in line and not ('wrapper.py logout' in line):
                parts = line.split()
                pid = parts[1]
                start_time = ' '.join(parts[8:9])
                wrapper_processes.append({
                    'pid': pid,
                    'time': start_time
                })
        
        return wrapper_processes
    except:
        return []

    def is_running():
    processes = get_instance_info()
    return len(processes) > 0

    def logout():
    processes = get_instance_info()

    if not processes:
        print(f'''
    {"-" * 120}

    No running instances found.
    To start a new instance:
    python3 wrapper.py


    {"-" * 120}
    ''')
        return False

    killed_count = 0
    failed_count = 0

    print(f"\n{'-' * 120}\n\nTerminating instances...")
    for proc in processes:
        try:
            pid = int(proc['pid'])
            os.kill(pid, signal.SIGKILL)
            killed_count += 1
            print(f"Terminated PID: {pid}")
        except ProcessLookupError:
            continue
        except PermissionError:
            failed_count += 1
            print(f"Permission denied for PID: {pid}")
        except Exception as e:
            failed_count += 1
            print(f"Failed to terminate PID {pid}: {str(e)}")

    if failed_count > 0:
        print(f'''
    Some processes could not be terminated due to permission issues.
    Try running with sudo:
    sudo python3 wrapper.py logout
    ''')
        return False

    if killed_count > 0:
        print(f"\nSuccessfully terminated {killed_count} instance{'s' if killed_count > 1 else ''}.")

    print(f"\n{'-' * 120}\n")
    return killed_count > 0

    def show_status():
    processes = get_instance_info()
    if not processes:
        print(f'''
    {"-" * 120}

    Status: Not running

    To start a new instance:
    python3 wrapper.py


    {"-" * 120}
    ''')
        return False

    status_message = f'''
    {"-" * 120}

    Status: Running
    Instance{"s" if len(processes) > 1 else ""}:'''

    for proc in processes:
        status_message += f'''
    PID: {proc['pid']}
    Started at: {proc['time']}'''

    status_message += f'''

    To terminate, run:
    python3 wrapper.py logout


    {"-" * 120}
    '''
    print(status_message)
    return True

    def get_credentials():
    login_message =f'''
    {"-" * 120}

    Login with Your Apple Music / iCloud or Apple ID
    To continue, please log in with your Apple ID credentials (email or phone number).

    Note: 
    If you have Two-Factor Authentication (2FA) enabled, a verification code will be sent to your trusted device.
    Enter the code to complete the login process.

    Forgot Your Password? 
    Visit Reset your password (https://iforgot.apple.com/password/verify/appleid) to reset it.

    Need Further Help?
    For any login issues, please visit Apple Support (https://support.apple.com/) for assistance.


    Username (add +86 prefix for Chinese mainland accounts): '''
    username = input(login_message)
    password = getpass.getpass("Password: ")
    print("\n")
    return f"{username}:{password}"

    def log_output(timestamp, message):
    with open("wrapper_log.txt", "a") as log:
        log.write(f"[{timestamp}] {message}\n")

    def handle_2fa():
    code = input(f"\n\nEnter the 2FA code: ")
    print("\n")
    if code == "" or len(code) != 6:
        return "000000" + "\n"  # Assume a wrong code to stop
    return code + "\n"

    def background_process(log_file):
    with open(os.devnull, 'r') as devnull:
        os.dup2(devnull.fileno(), sys.stdin.fileno())
    with open(log_file, 'a') as log:
        os.dup2(log.fileno(), sys.stdout.fileno())
        os.dup2(log.fileno(), sys.stderr.fileno())

    def read_output(process):
    m3u8_seen = False
    is_type_6 = False
    waiting_for_2fa = False
    response_type = None

    while True:
        line = process.stdout.readline()
        if not line and process.poll() is not None:
            break
        if line:
            timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            line = line.decode('utf-8', errors='replace').strip()

            if not m3u8_seen:
                print(f"[{timestamp}] {line}")

            log_output(timestamp, line)

            # Check for 2FA request
            if "2FA: true" in line:
                if not waiting_for_2fa:
                    waiting_for_2fa = True
                print(f"\n\nTwo-Factor Authentication (2FA) passcode required:\nA 2FA passcode has been sent to your devices using your preferred method.")
                code = handle_2fa()
                process.stdin.write(code.encode())
                process.stdin.flush()
                continue

            # Check for response type
            if "response type" in line.lower():
                try:
                    response_type = int(line.split("response type")[-1].strip())
                    if waiting_for_2fa and response_type == 0 or response_type == 4:
                        print("\n\n2FA verification failed.")
                        waiting_for_2fa = False
                    if response_type == 0:
                        login_failed_0 = f'''
    Login failed. Please check your credentials and try again.
    If the issue persist, please visit Apple Support (https://support.apple.com/) for further assistance.

    Aborting...


    {"-" * 120}
                        '''
                        print(login_failed_0)
                        sys.stdout.flush()
                        process.terminate()
                        sys.exit(1)
                    elif response_type == 4:
                        login_failed_4 = f'''
    Your account has been disabled for security reasons.
    If the issue persist, please visit Apple Support (https://support.apple.com/) for further assistance.

    Aborting...


    {"-" * 120}
                        '''
                        print(login_failed_4)
                        sys.stdout.flush()
                        process.terminate()
                        sys.exit(1)
                    elif response_type == 6:
                        is_type_6 = True
                except ValueError:
                    pass

            # Check for m3u8 message
            if "listening m3u8 request on" in line.lower():
                m3u8_seen = True

            # Check if both conditions are met
            if is_type_6 and m3u8_seen:
                if waiting_for_2fa:
                    print("\n\n2FA verified successfully.")
                    waiting_for_2fa = False
                print(f"\n\nLogin Succeed.\n\nInstance PID: {process.pid}\n\n{'-' * 120}")
                print("\nService is ready. Moving to background...\n")
                print("To check status: python3 wrapper.py status")
                print("To logout: python3 wrapper.py logout\n")
                sys.stdout.flush()
                time.sleep(0.5)  # Give time for messages to be printed

                # Start background process
                ctx = multiprocessing.get_context('spawn')
                p = ctx.Process(target=background_process, args=("wrapper_log.txt",))
                p.daemon = True
                p.start()

                # Clean up and exit
                process.stdin.close()
                process.stdout.close()
                process.terminate()
                os._exit(0)

            # Check for login failure explicitly
            if "login failed" in line.lower():
                if waiting_for_2fa:
                    print("\n\n2FA verification failed.")
                login_failed = f'''
    Login failed. Please try again later.
    If the issue persist, please visit Apple Support (https://support.apple.com/) for further assistance.

    Aborting...

    {"-" * 120}
                '''
                print(login_failed)
                sys.stdout.flush()
                process.terminate()
                sys.exit(1)

    def main():
    # Move the set_start_method inside main()
    if sys.platform != 'darwin':  # If not on macOS
        try:
            multiprocessing.set_start_method('spawn')
        except RuntimeError:
            pass  # Context already set, ignore the error
    # Command line argument handling
    if len(sys.argv) == 2:
        if sys.argv[1] == "logout":
            logout()
            return
        elif sys.argv[1] == "status":
            show_status()
            return
        else:
            print("Error: Invaild argument")
            sys.exit(1)
    elif len(sys.argv) > 2:
        print("Error: Invaild arguments")
        sys.exit(1)
    else:
        pass

    # Check if already running
    if is_running():
        print(f'''
    {"-" * 120}

    Error: An instance is already running.
    To check status:
    python3 wrapper.py status

    To start a new instance, please logout first:
    python3 wrapper.py logout

    {"-" * 120}
    ''')
        sys.exit(1)

    try:
        if not os.path.isdir("wrapper"):
            print("Error: 'wrapper' directory not found. Aborting...\n")
            sys.exit(1)

        os.chdir("wrapper")

        if not os.path.isfile("./wrapper"):
            print("Error: 'wrapper' executable not found. Aborting...\n")
            sys.exit(1)

        if not os.access("./wrapper", os.X_OK):
            print("Error: 'wrapper' is not executable. Aborting...\n")
            sys.exit(1)

        credentials = get_credentials()

        process = subprocess.Popen(
            ["./wrapper", "-D", "10020", "-M", "20020", "-L", credentials],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            bufsize=0,
            universal_newlines=False
        )

        output_thread = threading.Thread(target=read_output, args=(process,))
        output_thread.daemon = True
        output_thread.start()

        # Wait for a short time to allow the thread to process initial output
        time.sleep(0.1)

        # Wait for the process to complete or be terminated
        process.wait()

    except FileNotFoundError as e:
        print(f"Error: {str(e)}")
        sys.exit(1)
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
        main()
    ```
    \
    and store it to your root user directory.
    \
    2. Run the code and enter your Apple Music credentials.
    \
    3. Once the service is being moved to background. move on to Step 2 in the same terminal. 

**2. Run the `main.go` file with your preferred options:**
- To download the whole album:
```sh
go run main.go https://music.apple.com/us/album/whenever-you-need-somebody-2022-remaster/1624945511
```

- To download some selected songs in the album: 
```sh
go run main.go --select https://music.apple.com/us/album/whenever-you-need-somebody-2022-remaster/1624945511
```
Once prompted, enter your numbers separated by spaces.
- To download the entire playlist: 
```sh
go run main.go https://music.apple.com/us/playlist/taylor-swift-essentials/pl.3950454ced8c45a3b0cc693c2a7db97b
``` 
or 
```sh
go run main.go https://music.apple.com/us/playlist/hi-res-lossless-24-bit-192khz/pl.u-MDAWvpjt38370N
```

- To download all albums from an artist:
```sh
go run main.go https://music.apple.com/us/artist/taylor-swift/159260351 --all-album
```

- To download songs with Dolby Atmos® support: 
```sh
go run main.go --atmos https://music.apple.com/us/album/1989-taylors-version-deluxe/1713845538
```

and replace the URL above with your vaild Apple Music URL

### For Downloading Lyrics

1. Open Apple Music and log in.
2. Open the Developer tools, Click `Application -> Storage -> Cookies -> https://music.apple.com`.
3. Find the cookie named `media-user-token` and copy its value.
4. Paste the cookie value obtained in step 3 into the `config.yaml` and save it.
5. Start the script as usual.

---

**Note:** For detailed instructions, please refer to [step 3 in the original documentation](https://telegra.ph/Apple-Music-Wrapper-On-WSL1-07-21) (available in Simplified Chinese only).
