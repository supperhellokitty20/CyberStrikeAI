---
name: binary-mobile-reversing
description: >-
  APK/EXE/binaries: UniApp/DCloud/Flutter reversing, certificate pinning bypass, exported components, memory corruption exploit chains, IoT firmware. Use when reversing APK/EXE, UniApp/Flutter, native .so, or memory-corruption exploits.
metadata:
  tags: [penetration-testing, penetration-testing, red-team]
---

## APK / EXE / Binary Reversing

```
=== APK/EXE/Binaries ===
APK: apktool d / jadx | Manifest: check exported components/deeplink/debuggable | rg hardcoded keys + API addresses | .so strings/Ghidra | frida/objection dynamic
🚨UniApp/DCloud APK reversing (H5 hybrid apps, all business logic in JS not DEX):
  Identify: __UNI__XXXXXXX + assets/apps/ + uni-jsframework.js (NOT Flutter, no libapp.so) | manifest.json contains the "uni-app" field
  Core files: assets/apps/__UNI__XXX/www/app-service.js (800KB+ obfuscated JS = all business logic) | jadx can only see the shell (native plugin registration), the real logic is in JS
  JS deobfuscation (RC4 + string array rotation): extract the large a0G() array (10000+ elements) + the a0m(idx,key) decode function (base64→RC4) + rotation IIFE → assemble executable JS → batch decode with node → JSON dictionary (offset→plaintext)
    🔴Pitfall: the rotation IIFE may end with a comma (not a semicolon! it belongs to a larger expression) → node reports "Unexpected token" → extract by the actual terminator
  Encrypted config zlsioh.dat/dcloud3.dat: header (96B offset table: off40=block1 decompress/44=compress/60=block2 offset/64=block2 size/80=block3 offset)+block1 (zlib→DEX)+block2 (encrypted, contains the API domain list ← key, needs .so decryption)+block3 (zlib→AndroidX class mapping)
    zlib magic 78DA = unencrypted, decompress directly | no magic = encrypted block (key in .so .rodata/assembly immediates)
  .so string deobfuscation: extract even-position characters→reverse (e.g. "mAojcl.dubdFiHaebP.nwywfwb"→"www.baidu.com") | strings -n8 lib*.so to find long strings containing dots
  ⚠️Pitfall: x-separated IP strings in .so (e.g. "x111.230.69.120x118.126.105.164") = DCloud HTTPDNS nodes (NOT API backends! 140.205.11.x=Alibaba Cloud DNS, 111.230/118.126/106.52/42.193=Tencent Cloud) | resources.arsc errors are intentional anti-decompile protection (does not affect jadx) | fake PNGs in assets (1-8px) = integrity checks, not data | fcapp.run is only an APK download proxy
  Key API variables: $apiHost/$opHost (set dynamically at runtime, not hardcoded)/urlList/checkAvailableDomainList (HEAD domain/favicon.ico to test connectivity, 200-399=available)
  High-value endpoints: /app_init /user/gustRegister (guest registration without verification) /user/getOssSts (OSS temporary credentials!) /uploadFile2
  Config: manifest.json (appId/version/nativePlugins→wrs-httpserver=embedded HTTP service!) | supplierconfig.json (vivo/xiaomi/huawei/oppo appid) | dcloud_uniplugins.json (plugin manifest)
  ChengZi SDK (ChengZi Site Builder): the init3 endpoint returns XOR single-byte (key=0x96, see chengzi_decrypt.py) encrypted JSON→decrypt to get the real APK download link (fu field)+channel code
  Domain fallback: when $apiHost cannot be obtained statically→Android emulator (Waydroid/AVD)+tcpdump/mitmproxy to capture runtime DNS
  See also: references/uniapp-apk-reverse-engineering.md, references/uniapp-apk-reversing.md, scripts/js_rc4_deobfuscate.js, scripts/chengzi_decrypt.py
🚨Flutter APK quick reversing (no unpacking needed): hardening (MogoSec/Bangcle/360) protects the DEX layer, but Flutter's lib/arm64-v8a/libapp.so (Dart AOT) is usually not encrypted
  strings -n8 libapp.so|grep 'https\?://' → all hardcoded URLs/API domains/S3 addresses/CDN
  strings -n8 libapp.so|grep -E '^/' → API path matrix (/Member/Login, /Web/VideoList, /BBS/GetSTSToken, etc.)
  strings -n8 libapp.so|grep -iE 'key|secret|token|aws|bucket' → credential/key leakage
  assets/config_*.xml + assets/data_*.dat → encrypted config (may contain domains/API addresses) | .DS_Store → macOS developer info leak | assets/xinstall* → channel tracking SDK
🚨Native .so string deobfuscation (common pattern): obfuscated strings live in the .rodata section
  Common patterns: even-position character extraction+reverse (e.g. "mAojcl.dubdFiHaebP..."→take even positions→reverse=plaintext domain) | XOR constant | RC4+base64
  Identify: strings -n8 lib*.so to find 16-byte equal-length strings (AES key/IV)/strings containing dots (domains)/x-separated IP strings (HTTPDNS) | nm --dynamic to find obfuscated exported symbols (16-char random case)
  Verify: compare known strings ("classes.dex"/"io.dcloud.application") against the obfuscated version→reverse-engineer the algorithm→batch decode
🚨General mobile (regular apps that are not UniApp/Flutter):
  Certificate pinning bypass (capture HTTPS): objection android sslpinning disable | frida universal-unpinning | patch smali to remove pinning and repackage
  Exported component privilege escalation: Activity/Service/Provider/Receiver with exported=true in the Manifest → cross-app invocation with drozer/adb am start | ContentProvider SQLi/path traversal
  Deep link hijacking: scheme:// without validation→WebView loads arbitrary URL (XSS/file read) | parameters passed into intent→component hijacking
  Insecure storage: /data/data/pkg/ (plaintext token in shared_prefs/unencrypted db/logcat leakage) | world-readable sdcard
  WebView: addJavascriptInterface (<4.2 RCE)/file:// local file read/setAllowFileAccess | Static: MobSF one-shot | iOS: dump the decrypted binary (frida-ios-dump)+Ghidra to inspect Mach-O+objection dynamic
EXE/PE: file/strings | Ghidra/IDA decompile to find hardcoded values/encryption/network logic | x64dbg dynamic debugging | Vulnerabilities: overflow/format string/UAF/DLL hijacking
🚨Memory corruption exploit chain (crash→RCE): checksec to inspect mitigations → determine the primitive (stack overflow/UAF/format string=arbitrary read/write)
  → info leak to bypass ASLR → ROP chain (ROPgadget/pwntools ret2libc) → heap exploitation (tcache poison/__free_hook hijack)
  → arbitrary write to modify GOT/hook/vtable/exit_funcs → control-flow hijack | IoT firmware: binwalk -Me extract+qemu-user debugging (weak mitigations, often a straight stack overflow)
```
