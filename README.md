# NightRunners-Linux-Compat
Compatibility guide to run multiplayer mod for Night-Runners Private Alpha on Linux through Proton


This guide assumes you already have the game files for the latest private alpha versions (3.27 or 3.28) and are running the game through the Flatpak version of Steam.

I have not tested this with versions of Steam from distro package managers, but I imagine that most of the steps should work if you change the file paths and ignore Flatpak-specific steps.

Table of Contents

1. Setting Up Steam and Proton

2. Installing the Mods

3. Understanding the Steam Detection Issue

4. Fixing Steam Profile Detection

A. Calculate Your Steam Account ID

B. Back Up and Edit the Registry

C. Link the Steam Account Configuration

D. Link the Profile Picture Cache

5. Launching the Game

6. Troubleshooting

1. Setting Up Steam and Proton

If you have not already added the game to Steam, get the game files and extract them to wherever you want the game to be on your hard drive.

Go to Steam and choose Add a Game in the bottom-left corner, then Add a Non-Steam Game.

Click Browse and select Night-Runners Private Alpha.exe. If Steam shows the file path as something like /run/user/1000, you may have to use a tool like Flatseal and manually allow Steam to access the folder containing the game.

You can do this by going to the Filesystem section, then Other files, clicking the Add folder button, and pasting in the path to your game folder.

For example, I put ~/NightRunnersFiles/Night Runners Private Alpha v3.27/ in mine because this is where I have my installation. Replace the example path with the rest of the path to your own directory.

After adding the game to Steam and configuring Flatpak permissions with Flatseal if required, right-click on it in Steam and select Properties. You should see the game .exe and the path to the folder location under Target and Start In, respectively.

To ensure that Proton will be able to read the mod files, add this to the Launch Options field:

WINEDLLOVERRIDES="winhttp.dll=n,b" %command%

Although I used GE-Proton for my testing, Proton Experimental also seemed to work fine when I tested it. Go to Compatibility in the same Properties window and select Proton Experimental.

If Proton Experimental has issues, then I recommend using a tool like ProtonUp-Qt to add the latest version of GE-Proton to your Steam install.

At this point, you should have the bare minimum so that if you launch the game without mods, it will run.

2. Installing the Mods

Now, we will install the mods.

Download the latest .dll of NRMM from https://github.com/AlexR32/nrmm (I used version 1.0.8 for my testing), then download the latest ZIP of NRMP from https://github.com/NR-MP/NRMP-MOD/releases that corresponds to your game version(i.e. 3.27).

Drag both downloads to the game folder. Extract all files from the NRMP ZIP into the game folder. You should see a BepInEx folder now, as well as several other new folders and files.

Delete the ZIP, then ctrl + x the NRMM .dll file. Go to BepInEx/plugins and paste it there. You should also see the NightRunnersMP directory there if you did everything right.

Now, the game will work with mods. If you see the option to launch multiplayer in the game's main menu but are told "Open steam to play multiplayer", then you did everything right as far as the mod installation goes.

3. Understanding the Steam Detection Issue

This is broken because of how the multiplayer mod handles getting Steam profile information for player cards in online mode.

In normal games that are installed through Steam, the game can use a SteamAPI.Init() call to read profile information from Steamworks.

Being a non-Steam game does not inherently prevent a game from using Steamworks, but the private alpha does not implement Steamworks integration because it is meant to be played through Itch.

The dev works around this on a normal Windows install by reading the username and profile picture from the Windows registry and Steam's configuration files in src/NightRunnersMP/Steam.cs, with the following comments:

// no steamworks in the game, so read what the steam client leaves around:
// account in the registry, name and pic in its files

From HKCU\Software\Valve\Steam\ActiveProcess, the game reads:

ActiveUser - Steam account ID.

pid - the Steam process ID.

From HKCU\Software\Valve\Steam, the game reads:

SteamPath - where Steam is installed.

It then checks whether the process identified by pid is running and reads config/loginusers.vdf to find your Steam display name.

It also constructs a path to the profile picture using config/avatarcache/[SteamID64].png.

If any of those checks fail, Steam.Read() returns null, and the multiplayer mod logs:

Steam isn't running, staying offline

The Proton prefix is missing the ActiveUser registry value, and the Steam compatibility directory does not contain the actual loginusers.vdf or avatar cache. The native Steam installation does contain these files.

So now what we must do is find a way to provide the equivalent registry information and files inside the Proton environment.

To do this, we will add the missing registry value and create symbolic links between some Steam files outside of Proton and their expected locations inside the Proton environment. This is better than copying the files from Steam into the Proton environment, as the links will point directly to the native Steam files and automatically reflect changes made outside Proton.

4. Fixing Steam Profile Detection

Open a terminal and define the Steam installation and game prefix paths:

STEAM="$HOME/.var/app/com.valvesoftware.Steam/.local/share/Steam"
COMPAT="$STEAM/steamapps/compatdata/YOUR_GAME_PREFIX_ID"
WINSTEAM="$COMPAT/pfx/drive_c/Program Files (x86)/Steam"
REG="$COMPAT/pfx/user.reg"

Replace YOUR_GAME_PREFIX_ID with the compatibility prefix ID for your game.

You can find this by sorting the folders in compatdata by date created. If you just ran this version of NIGHT-RUNNERS Private Alpha for the first time, it will likely be the newest folder. Alternatively, you can manually search through folders if your install is older.

To do this, open a folder with a bunch of numbers (the Steam application ID) as its name, then go to:

pfx/drive_c/users/steamuser/AppData/LocalLow/PLANET JEM SOFTWARE

If this directory exists and you see a NIGHT-RUNNERS PRIVATE ALPHA folder, then the aforementioned bunch-of-numbers folder is the ID to use for YOUR_GAME_PREFIX_ID.

It is also where your game's SaveFile.es3 is located if you want to edit or back up your save.

[!NOTE]
Do not assume that the prefix ID will be the same for all users. Non-Steam game shortcuts receive their own Proton compatibility prefixes, and the ID depends on your Steam installation.

A. Calculate Your Steam Account ID

Steam uses a 17-digit SteamID64, but NRMP calculates this ID using the 32-bit ActiveUser value from the Windows registry:

SteamID64 = 76561197960265728 + ActiveUser

Run:

ID=$(grep -oE '[0-9]{17}' "$STEAM/config/loginusers.vdf" | head -n1)

printf '%08x\n' "$((ID - 76561197960265728))"

This prints the hexadecimal account ID needed by the registry.

[!WARNING]
This command assumes the first account listed in loginusers.vdf is the account you want to use. If you have multiple Steam accounts, make sure you select the correct SteamID64 instead.

It is not recommended to share your SteamID64 publicly, as it is a permanent account identifier that can be used to track your account even if you change your display name.

B. Back Up and Edit the Registry

Close the game, but leave Steam running.

Back up the registry in case these changes break the game:

cp -a "$REG" "$REG.bak-nrmp"

Insert the missing ActiveUser value and replace YOUR_HEX_ID with the hexadecimal value from the previous step:

perl -0pi -e '
  s/^(\[Software\\\\Valve\\\\Steam\\\\ActiveProcess\][^\n]*\n)/
    $1 . "\"ActiveUser\"=dword:YOUR_HEX_ID\n"
/me or die "ActiveProcess registry key not found\n"
' "$REG"

Leave the existing PID and Steam DLL entries as they are and do not change them.

[!WARNING]
The existing PID is a Windows/Proton-side process identifier, not necessarily the same as the Linux Steam process ID. Do not replace it with the PID shown by Linux commands such as pgrep.

C. Link the Steam Account Configuration

Link the Steam account configuration inside the Proton environment to the Steam configuration in the Flatpak Steam install.

Run:

ln -s "$STEAM/config/loginusers.vdf" \
  "$WINSTEAM/config/loginusers.vdf"

This creates a symbolic link rather than copying the file. NRMP can now access the actual Steam account configuration through the Windows-style Steam directory in the Proton prefix.

D. Link the Profile Picture Cache (Optional, multiplayer works fine without this and I initially ran the mod without fixing pfp).

To enable Steam profile picture in NRMP, run:

ln -s "$STEAM/config/avatarcache" \
  "$WINSTEAM/config/avatarcache"

The avatar cache contains the PNG files NRMP expects to find using your SteamID64. Linking the entire directory means Steam can update its cached profile pictures when you change it without requiring you to copy them manually.

5. Launching the Game

Launch the game through Steam.

NRMP should now recognize your Steam username and account, and multiplayer should be available by pressing M on the main menu.

If you want to confirm that Steam profile detection succeeded, check:

[Game Root]/BepInEx/LogOutput.log

You should see a message similar to:

Steam name [your username], picture True

If the username is detected but picture is False, check that the avatar cache link exists and that the corresponding PNG file is present.

6. Troubleshooting

If the registry edit or symbolic-link commands fail, check that you selected the correct Proton prefix and that the corresponding directories exist.

Some other things to check:

Make sure Steam is running and that you are logged into the intended account.

Make sure the ActiveUser value was inserted into the ActiveProcess registry section, not another Steam registry section.

Make sure you used the hexadecimal account ID, not the decimal value.

Make sure the loginusers.vdf link points to the native Steam configuration file.

If you have multiple Steam accounts, verify that the account ID used for ActiveUser corresponds to the account you are currently using.

If the game still reports that Steam isn't running, inspect BepInEx/LogOutput.log and verify the registry values and symbolic links before changing anything else.

If you need to undo the registry modification, close the game and restore your backup:

cp -a "$REG.bak-nrmp" "$REG"

This restores the registry to its state before the workaround, but will not remove the symlinks.

If you have specific issues or questions, ping me in the NRMC or NRMP Discord server.

Note: This workaround is based on the current implementation of NRMP's Steam detection. Future updates to NRMP, Proton, or Steam may change the relevant paths or behavior. If a future update breaks this setup, check the mod's source code and logs before assuming the same workaround still applies.
