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
    -h|--help           display this help and exit
    -q|--quiet          show no informational output
    -p|--password       provide a password for encryption/decryption
and one of the following commands
    -g|--generate <len> generate a random string of provided length and exit
    -e|--encrypt        encrypt the @savv directives in the file
    -r|--reverse        reverse the encryption to view in a format to easily edit, and re-encrypt
    -v|--view           view the decrypted values in a format that can be used in scripts

When encrypting single variables that are prefixed with @savv:... are encrypted. Examples are
    export VAR1=@savv:encrypt:secret1
    export VAR3=@savv:generate
    export VAR4=@savv:generate:64
The lines will be replaced with something like:
    VAR1=$(decrypt_str ...)

The decrypt_str is a function with the following defintion
    decrypt_str() {
        echo $( echo  | openssl aes-256-cbc -d -a -A -pbkdf2 -pass pass:$SAVV_PASSWORD )
    }

When using --view, the file is shown on the output with all $(decrypt_str ...) reversed

When using --reverse, the file is modified using a special @savv:orig:...:new:... syntax
One can then edit the file and change some of the secrets after the new: tag
When re-encrypting using --encrypt, the orig value in orig: is used if the secret was not changed
Normally openssl will encrypt secrets with a random salt so that each time the encrypted value is different.
This feature using the orig value makes it more clear in version control systems which lines are changed.
```

## Goals
- work well with version control system like git
- be able to edit a file with several secret values
- automatically generate new passwords
- easily reversibly decrypt to edit secrets, while keeping unchanged secrets unchanged

The last part is tricky.
If one would just decrypt the file, edit it and re-encrypt, the re-encryption would change the encrypted result even if the secret is unchanged.
This is not desirable for version control systems like git.
