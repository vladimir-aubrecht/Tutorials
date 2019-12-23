# Blizzard 2FA for 1Password
For those who were trying to use 1Password for 2FA in Blizzard services, here is a tutorial how to get 1Password's scannable QR code.

## MacOS tutorial

### Install Brew
```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
export PATH="/usr/local/opt/python/libexec/bin:$PATH"
```

### Install Python 3
```
brew install python
```

### Install BNA*
```
git clone https://github.com/jleclanche/python-bna.git
```

### Install QRCode
```
python -m pip install qrcode
```

#### Generate QRCode
```
python ./python-bna/bin/bna --restore <serial_code> <restore_code>
python ./python-bna/bin/bna --otpauth-url
qr “<otp_url>”
```
