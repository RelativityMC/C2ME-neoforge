<img width="200" src="https://github.com/RelativityMC/C2ME-fabric/raw/ver/1.17/src/main/resources/assets/c2me/icon.png" alt="C2ME icon" align="right">
<div align="left">
<h1>C^2M-Engine</h1>

[![Discord](https://img.shields.io/discord/756715786747248641?logo=discord&logoColor=white)](https://discord.gg/Kdy8NM5HW4)
<h3>A mod designed to improve the chunk performance of Minecraft.</h3>
</div>

## So what is C2ME?
C^2M-Engine, or C2ME for short, is a mod designed to improve the performance of chunk generation, I/O, and loading. This is done by taking advantage of multiple CPU cores in parallel. For the best performance it is recommended to use C2ME with [Lithium](https://github.com/CaffeineMC/lithium-fabric) and [Starlight](https://github.com/Spottedleaf/Starlight).

## What does C2ME stand for?
Concurrent chunk management engine, it's about making the game better threaded and more scalable in regard to world gen and chunk io performance.

## So what is C2ME not?
**C2ME is currently in alpha stage and pretty experimental.**  
Although it is usable in most cases and tested during build time, it doesn't mean that it is fully stable for a production server.  
So backup your worlds and practice good game modding skills.

## Downloads
Modrinth: https://modrinth.com/mod/c2me-neoforge  
CurseForge: https://www.curseforge.com/minecraft/mc-mods/c2me-neoforge

## Support
Our issue tracker: [link](https://github.com/RelativityMC/C2ME-neoforge/issues)  
Our discord server: [link](https://discord.gg/Kdy8NM5HW4)

## Building and setting up
JDK 22+, Clang 18+ are required to build C2ME

#### Initial setup
Run the following commands in the root directory:

```
git submodule update --init
./build.sh up
./build.sh patch
```

#### Creating a patch
See [CONTRIBUTING.md](CONTRIBUTING.md) for more detailed information.

#### Compiling
Use the command `./build.sh build`. Compiled jars will be placed under `C2ME-fabric-patched/build/libs/`.

## License
License information can be found [here](/LICENSE).

