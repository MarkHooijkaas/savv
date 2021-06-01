# savv: Secure Any Variable reVersibly
savv can secure (encrypt) variables using just bash and openssl for generic purposes

savv has the following features:
- easy to use
- encrypt secrets annotated with @savv:encrypt:<secret>
- generate and encrypt new secrets with @savv:generate:<length>
- view encrypted secrets
- reverse encryption (decrypt) to edit secrets, while keeping unchanged secrets unchanged

## Syntax
```
Usage: savv [options] <file>...

The purpose of savv is working with encrypted (shell) variables
Options can be:
    -h|--help               display this help and exit
    -q|--quiet              show no informational output
    -p|--password           provide a password for encryption/decryption
    -g|--generate[:<len>]   generate a random string of provided length (default 32) and exit
    -e|--encrypt            encrypt the @savv directives in the file
    -r|--reverse            reverse the encryption to view in a format to easily edit, and re-encrypt
    -v|--view               view the decrypted values in a format that can be used in scripts

If none of the --generate, --reverse or --export options are given, the default mode is to encrypt.
In this mode single variables that are prefixed with @savv are encrypted. Examples are
    export VAR1=@savv:encrypt:secret1
    export VAR3=@savv:generate
    export VAR4=@savv:generate:64
A label of @savv:generate will generate a password of length 32
The lines will be replaced with something like:
    VAR1=$(decrypt_str ...)

The decrypt_str is a function with the following defintion
    decrypt_str() {
        echo $( echo  | openssl aes-256-cbc -d -a -A -pbkdf2 -pass pass:$SAVV_PASSWORD )
    }
```

## Goals
- work well with version control system like git
- be able to edit a file with several secret values
- automatically generate new passwords
- easily reversibly decrypt to edit secrets, while keeping unchanged secrets unchanged

The last part is tricky.
If one would just decrypt the file, edit it and re-encrypt, the re-encryption would change the encrypted result even if the secret is unchanged.
This is not desirable for version control systems like git.
