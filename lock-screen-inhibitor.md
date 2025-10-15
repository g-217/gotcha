### How to keep your machine awake for a fixed time for running tests etc.
### MacOS: `caffeinate` 
- `caffeinate <comand>` will keep the machine awake for duration of the command.
- `caffeinate -t <numbe of seconds>` will do same for specified duration.
- `caffeinate -d` will do same till `Ctrl+C` is pressed.

### Linux: `systemd-inhibit`
- `systemd-inhibit --what=sleep --what=shutdown my_long_running_script.sh` 
