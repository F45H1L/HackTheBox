# Scalped Platform — HackTheBox

### Challenge Link: https://app.hackthebox.com/challenges/Scalped%2520Platform?tab=play_challenge

## Challenge

The challenge presents an abandoned satellite On-Board Computer (OBC) that drops into a U-Boot shell because its normal boot devices are unavailable.

The objective is to recover configuration remnants from the OBC and retrieve the flag.

---

## 1. Connect to the Target

From Kali:

```bash
nc <TARGET-IP> <TARGET-PORT>
```

The system drops into a U-Boot prompt:

```text
=>
```

The boot process shows:

```text
Loading Environment from EEPROM...
*** Warning - EEPROM erased, fallback to other I2C bus
Loading Environment from EEPROM...
*** Warning - bad CRC, using default environment
```

This suggests that an EEPROM may contain useful remnants.

---

## 2. Enumerate Available U-Boot Commands

Run:

```text
help
```

The interesting commands are:

```text
i2c
md
printenv
```

The I²C subsystem is especially interesting because the boot log explicitly mentions EEPROM.

---

## 3. Enumerate I²C Buses

Run:

```text
i2c bus
```

Output:

```text
Bus 0: I2C bus 0
Bus 1: I2C bus 1
```

Check bus 0:

```text
i2c dev 0
i2c probe
```

Output:

```text
Valid chip addresses: 18 50
```

Check bus 1:

```text
i2c dev 1
i2c probe
```

Output:

```text
Valid chip addresses: 18 50 68
```

The device at `0x50` is consistent with an I²C EEPROM.

---

# 4. Inspect EEPROM on Bus 0

Select bus 0:

```text
i2c dev 0
```

Check the EEPROM offset length:

```text
i2c olen 50
```

Output:

```text
1
```

Initially reading the EEPROM:

```text
i2c md 50 0 64
```

returns:

```text
ff ff ff ff ff ff ff ff ...
```

So the EEPROM on bus 0 is empty at the beginning.

However, the challenge may contain data at another offset.

---

# 5. Read the Bus 1 EEPROM

Switch to bus 1:

```text
i2c dev 1
```

The EEPROM uses a larger address space, so read it using a 2-byte offset:

```text
i2c md 50 0x800.2 0x100
```

Part of the output contains:

```text
0800: 00 e0 9f 55 61 72 63 68 3d 63 68 61 6e 67 65 64
0810: 00 62 61 75 64 72 61 74 65 3d 31 31 35 32 30 30
0820: 00 62 6f 61 72 64 3d 66 6c 61 74 2d 63 75 62 65
0830: 73 61 74 2d 6f 62 63 31 00 62 6f 61 72 64 5f 6e
0840: 61 6d 65 3d 66 6c 61 74 2d 63 75 62 65 73 61 74
0850: 2d 6f 62 63 31 00 62 6f 6f 74 5f 74 61 72 67 65
0860: 74 73 3d 6d 6d 63 31 00 62 6f 6f 74 63 6d 64 3d
0870: 62 6f 6f 74 66 6c 6f 77 20 73 63 61 6e 20 2d 6c
0880: 62 00 63 70 75 3d 58 43 37 5a 30 31 30 00 65 6e
0890: 63 72 79 70 74 65 64 5f 66 6c 61 67 3d 63 30 34
08a0: 61 62 62 66 30 34 36 64 65 39 33 36 65 36 65 35
08b0: 36 61 64 66 39 62 35 39 64 35 62 62 62 30 63 65
08c0: 39 32 65 66 64 39 30 33 36 36 34 30 65 31 38 38
08d0: 37 32 62 34 36 31 61 39 36 61 31 39 33 00 66 64
08e0: 74 5f 61 64 64 72 3d 30 78 34 30 30 30 30 30 30
08f0: 30 00 66 64 74 63 6f 6e 74 72 6f 6c 61 64 64 72
```

These hexadecimal bytes represent ASCII text.

---

# 6. Convert the Hexadecimal Bytes

Instead of manually translating every byte, use Kali's `xxd` tool.

Copy the hexadecimal bytes into a file:

```bash
nano hex.txt
```

For example:

```text
00e09f55617263683d6368616e6765640062617564726174653d31313532303000626f6172643d666c61742d637562657361742d6f62633100626f6172645f6e616d653d666c61742d637562657361742d6f62633100626f6f745f746172676574733d6d6d633100626f6f74636d643d626f6f74666c6f77207363616e202d6c62006370753d5843375a30313000656e637279707465645f666c61673d63303461626266303436646539333665366535366164663962353964356262623063653932656664393033363634306531383837326234363161393661313933006664745f616464723d3078343030303030303000666474636f6e74726f6c61646472
```

Save the file and convert the hexadecimal representation back into bytes:
a
```bash
xxd -r -p hex.txt
```

Output:

```text
��Uarch=changedbaudrate=115200board=flat-cubesat-obc1board_name=flat-cubesat-obc1boot_targets=mmc1bootcmd=bootflow scan -lbcpu=XC7Z010encrypted_flag=c04abbf046de936e6e56adf9b59d5bbb0ce92efd9036640e18872b461a96a193fdt_addr=0x40000000fdtcontroladdr
```

Therefore the Encrypted Flag:

```text
encrypted_flag =
c04abbf046de936e6e56adf9b59d5bbb0ce92efd9036640e18872b461a96a193
```

---

# 7. Recover the AES Key and IV

Switch back to bus 0:

```text
i2c dev 0
```

Read from offset `0x100` using a 2-byte address:

```text
i2c md 50 0x100.2 0x100
```

Again the EEPROM contains hexadecimal bytes:
```text
0100: 41 45 53 5f 4b 45 59 3d 37 32 31 66 35 61 33 66
0110: 62 32 63 34 62 61 64 37 64 64 63 35 37 39 36 61
0120: 62 31 32 39 63 64 64 35 64 39 38 36 38 35 65 63
0130: 65 65 33 32 33 35 35 36 66 66 35 31 39 33 32 31
0140: 65 62 30 36 39 61 37 64 0a 49 56 3d 30 37 66 65
0150: 61 35 61 31 37 36 32 63 66 62 30 35 36 61 65 61
0160: 34 63 38 63 65 38 33 31 37 33 64 39 0a 41 45 53
0170: 5f 4d 4f 44 45 3d 43 42 43 0a ff ff ff ff ff ff
0180: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
0190: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
01a0: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
01b0: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
01c0: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
01d0: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
01e0: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
01f0: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
```

Similary, these can also be manually converted using Kali's `xxd`.

For example, put the relevant hexadecimal bytes into a file:

```bash
nano aes_hex.txt
```

For Example:
```text
4145535f4b45593d373231663561336662326334626164376464633537393661623132396364643564393836383565636565333233353536666635313933323165623036396137640a49563d30376665613561313736326366623035366165613463386365383331373364390a4145535f4d4f44453d4342430affffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```

Then:

```bash
xxd -r -p aes_hex.txt
```

This converts the hexadecimal byte representation into readable ASCII:
```text
AES_KEY=721f5a3fb2c4bad7ddc5796ab129cdd5d98685ecee323556ff519321eb069a7d
IV=07fea5a1762cfb056aea4c8ce83173d9
AES_MODE=CBC
��������������������������������������������������������������������������������������������������������������������������������������
```

Recovered values:

```text
AES_KEY=721f5a3fb2c4bad7ddc5796ab129cdd5d98685ecee323556ff519321eb069a7d
IV=07fea5a1762cfb056aea4c8ce83173d9
AES_MODE=CBC
```

The key is 64 hexadecimal characters, meaning it is a **256-bit AES key**.

Therefore the encryption is:

```text
AES-256-CBC
```

---

# 8. Decrypt the Flag

Use OpenSSL:
```bash
echo '<PLACEHOLDER-FOR-ENCRYPTED-FLAG>' \
| xxd -r -p \
| openssl enc -d -aes-256-cbc \
-K '<PLACEHOLDER-FOR-AES-KEY>' \
-iv '<PLACEHOLDER-FOR-IV>'
```

From my Example:

```bash
echo 'c04abbf046de936e6e56adf9b59d5bbb0ce92efd9036640e18872b461a96a193' \
| xxd -r -p \
| openssl enc -d -aes-256-cbc \
-K '721f5a3fb2c4bad7ddc5796ab129cdd5d98685ecee323556ff519321eb069a7d' \
-iv '07fea5a1762cfb056aea4c8ce83173d9'
```

The plaintext is:

```text
HTB{33pr0m_d035n7_f0rg37!}
```

---

# Flag

```text
HTB{33pr0m_d035n7_f0rg37!}
```

---

## Key Takeaways

* The target was running **U-Boot** rather than a normal operating system.
* The boot messages revealed that U-Boot was attempting to load its environment from **EEPROM**.
* I²C enumeration revealed EEPROM devices at address `0x50`.
* The EEPROM on **bus 1** contained a stale U-Boot environment and the encrypted flag.
* The EEPROM on **bus 0** contained the AES encryption parameters.
* `xxd -r -p` can be used in Kali to manually convert hexadecimal byte representations into ASCII.
* The recovered ciphertext was encrypted using **AES-256-CBC**.
* OpenSSL was used to decrypt the ciphertext and recover the flag.