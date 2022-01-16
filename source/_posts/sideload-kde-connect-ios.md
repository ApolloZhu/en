---
title: Sideload KDE Connect iOS
tags:
  - KDE Connect
categories:
  - Programming
date: 2022-01-15 21:00:42
---

{% note success %}

For general audience, please use the official App Store/[TestFlight](https://testflight.apple.com/join/vxCluwBF) releases.

{% endnote %}

<!-- more -->

{% note warning %}

This is intended for contributors outside of K Desktop Environment e.V. development group to test their code on a physical device before opening a merge request. This process should theoretically work even if you don't have a paid Apple Developer account, though I've never tested.

{% endnote %}

{% note danger %}

THE INSTRUCTION IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE INSTRUCTION OR THE USE OR OTHER DEALINGS IN THE INSTRUCTION.

{% endnote %}

1. Download KDE Connect iOS source code from KDE Invent
2. Open `KDE Connect.xcodeproj` in Xcode
   1. In `Build Settings` for target `KDE Connect`, delete `Code Signing Entitlements` (but do not delete the entitlements file)
   2. In `Singing & Capabilities` for target `KDE Connect`, change the `Team` and `Bundle Identifier` so the project builds for `Any iOS Device`.
   3. In `Products` folder, right click on `KDE Connect` (which shouldn't be red) and select `Show In Finder`
2. In terminal, navigate to the folder containing `KDE Connect.xcodeproj`
3. In terminal, type the following commands ***line by line*** (other than comments starting with `#`). See [References](#references) section about what each of these lines do:
   ```sh
   build=#drag the KDE Connect.app here from Finder
   security cms -D -i "$build/embedded.mobileprovision" > provision.plist
   name=$(basename "`pwd`")
   /usr/libexec/PlistBuddy -x -c 'Print:Entitlements' provision.plist > entitlements.plist
   # Note that extra ' is added because name contains whitespace in path
   /usr/libexec/PlistBuddy -x -c "Merge '$name/$name.entitlements'" entitlements.plist
   # Replace `YOUR NAME (TEAM)` with the actual value of your Apple Development Certificate name (found in KeyChain Access)
   codesign -d --entitlements entitlements.plist -f -s "Apple Development: YOUR NAME (TEAM)" $build
   ```
4. Place the `KDE Connect.app` in a folder named `Payload`, zip the `Payload` folder, and (re)name it `KDE Connect.ipa`
5. Install self-signed `KDE Connect.ipa` on your own physical device for testing (requires jailbreaking or apps like Sideloadly)

## References

- [CarPlay apps without Apple’s blessings on a real car](https://fotidim.com/carplay-apps-without-entitlements-in-an-actual-car-37a708758262)
- [How to convert *app to *ipa](https://gist.github.com/bananita/8039021)
