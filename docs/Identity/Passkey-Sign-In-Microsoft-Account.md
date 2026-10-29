# Sign in to a Microsoft Account with a Passkey

How to create a passkey for a **personal Microsoft account** (Outlook.com, Hotmail, Xbox, OneDrive personal) and use it to sign in without a password.

## What a passkey is

A passkey replaces your password with a key pair. The private key stays on your device, phone, password manager or security key. You unlock it with the same face, fingerprint or PIN you use to unlock the device.

| | Password | Passkey |
|---|---|---|
| Can be phished | Yes | No. It only works on the real Microsoft sign-in site |
| Can be reused or leaked | Yes | No. Each site gets its own key |
| What you type | Password + code | Nothing. Just face, fingerprint or PIN |

## Before you start

- A device that can store a passkey:
    - **Windows 10/11** with Windows Hello (face, fingerprint or PIN)
    - **iPhone / iPad / Mac** (iCloud Keychain) or **Android** (Google Password Manager)
    - A **FIDO2 security key** (for example YubiKey)
    - A password manager that supports passkeys (Microsoft Password Manager, 1Password and others)
- An up-to-date browser (Edge, Chrome, Safari or Firefox).
- Bluetooth turned on if you'll save the passkey to a phone while signing in on a PC.

## Create the passkey

Do this **on the device where you want the passkey stored**.

1. Go to [account.live.com/proofs/manage](https://account.live.com/proofs/manage) (**Advanced security options**) and sign in.
2. Select **Add a new way to sign in or verify**.
3. Select **Face, Fingerprint, PIN, or Security Key**.
4. Follow the prompt from your device. Select **Continue** / **Create** to save in the suggested place, or **Change** / **Save another way** to choose:

| Save location | Notes |
|---|---|
| Windows device (Windows Hello) | Stored on this PC only. May not be offered if you already saved a passkey to a synced password manager. |
| Password manager | Synced across your devices (Microsoft Password Manager, iCloud Keychain, Google Password Manager, ...). |
| iPhone, iPad or Android | Scan the QR code with the phone's camera or the Microsoft Authenticator QR scanner. May need Bluetooth. |
| Security key | Insert or tap the key, then set or enter its PIN. |

5. Confirm with your face, fingerprint or PIN. The passkey now appears under **Ways to prove who you are**.

!!! tip
    Create a second passkey on another device (for example your phone) so you are not locked out if you lose the first one.

## Sign in with the passkey

1. Go to a Microsoft site such as [outlook.com](https://outlook.com) and select **Sign in**.
2. Either enter your email and select **Next**, or select **Sign-in options** / **Sign in with a passkey** without typing anything.
3. Choose **Face, fingerprint, PIN or security key** if asked.
4. Unlock with your face, fingerprint or PIN, or scan the QR code with your phone if the passkey lives there.

You are signed in without a password.

## Go fully passwordless (optional)

Once a passkey or Microsoft Authenticator is set up, you can remove the password from the account:

1. Open [Advanced security options](https://account.live.com/proofs/manage).
2. Under **Additional security**, turn **Passwordless account** on and follow the prompts.

!!! warning
    Keep at least two sign-in methods (for example a passkey and Authenticator) before removing the password.

## Troubleshooting

| Symptom | Fix |
|---|---|
| No passkey option is offered | Update the browser and OS. In private/incognito windows some password managers are disabled. |
| "Windows Hello isn't available" | Set up face, fingerprint or PIN in **Settings > Accounts > Sign-in options** first. |
| Phone QR code never connects | Turn on Bluetooth on both devices and keep them close together. |
| Passkey works on one device only | It was saved to Windows Hello (local). Create another passkey in a synced password manager or on your phone. |
| Passkey was deleted or the device was lost | Remove it from **Advanced security options** and sign in with another method. |
| Work or school account | Use [mysignins.microsoft.com/security-info](https://mysignins.microsoft.com/security-info) > **Add sign-in method** > **Passkey**. Your organisation must allow it. |

## References

- [Create and save a passkey - Microsoft Support](https://support.microsoft.com/en-us/accounts-billing/security/create-save-passkey)
- [Signing in with a passkey - Microsoft Support](https://support.microsoft.com/en-us/account-billing/signing-in-with-a-passkey-09a49a86-ca47-406c-8acc-ed0e3c852c6d)
