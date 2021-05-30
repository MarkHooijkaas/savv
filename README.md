# savv and savvy
This project contains two programs:
- savv: securing variable using bash and openssl for generic purposes
- savvy: securing variables for ansible vault

Originaly only savvy was developed, specifically for use in a ansible based project.
For a later project savv was added, on the same principles of being reversible.
savv is more lightweight (only requires bash and openssl) and can be used for various applications such as bash shell scripts.
savvy needs python and some ansible libraries, and is only usuable for ansible.

The both have similar features (savv syntax shown):
- encrypt secrets annotated with @savv:encrypt:<secret>
- generate and encrypt new secrets with @savv:generate:<length>
- view encrypted secrets
- easily reversibly decrypt to edit secrets, while keeping unchanged secrets unchanged

The last part is tricky.
If one would just decrypt the file, edit it and re-encrypt, the re-encryption would change the encrypted result even if the secret is unchanged.
This is not desirable for version control systems like git.

## SAVV: Secure Any Variable reVersibly
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

## SAVVY: Single Ansible Vault Variable encrYpt/decrYpt

## Installing savvy
The quick and dirty way to install the latest version of savvy:
```
# get the savvy python file
wget https://raw.githubusercontent.com/MarkHooijkaas/savvy-tools/master/savvy

# set your vault password in en environment variable where savvy can find it:
export VAULT_PASS
read -s VAULT_PASS

# run savvy
./savvy --help
```
Prerequisite is that Python and ansible vault are installed.

## Using savvy
This set of tools is meant to use single variables in a normal unencrypted vars file that are encrypted with ansible-vault.
Ansible playbooks support the use of these variables, but the tooling to encrypt and decrypt these vars can be better.

The tools are:
* `savvy decrypt`: decrypt all `!vault` variables in a file with special @savvy marker
* `savvy split` :  decrypt all `!vault` variables in a separate mergefile
* `savvy encrypt`: encrypt all `@savvy` annotated variables in a file
* `savvy merge`: encrypt all variables from a mergefile into the main file (not implemented yet)
* `savvy view`: show all encrypted variables in a file
* `savvy censor`: read stdin and replace all secrets with a censored value
* `savvy edit`: decrypt, edit, and re-encrypt (not implemented yet)

All commands can be abbreviated with the first letter or letters, for example `savvy d`, `savvy de`, `savvy dec`, `savvy decr` will all decrypt the file. Since both `encrypt` and `edit` start with the letter `e` one must at least use `savvy ed` to edit a file. `savvy e` will encrypt the file

## Goals
* work well with version control system like git
* be able to edit a file with several secret values
* support single and multi line vault formats (not working yet)
* automatically generate new passwords

## Encrypting
To encrypt some variables in a file, these can be marked with a @savvy annotation as follows:
```
@savvy: some_password: BadPassword123
```
`savvy encrypt` will encrypt all the variables with this annotation (after trimming whitespaces after the colon and at the end of the line).

## Decrypting and re-encrypting
One of the main targets is to be able to simply edit the encypted variables in a file and the re-encypt them, where the encrypted string is not changed if the value is not changed. This is important in order to store the values in a version control system like git. In order to do this savvy-decrypt will keep the original encrypted value, next to the decrypted value. When encrypting with savvy-encrypt this will recognize if the value is changed, and if it is not modified reuse the old encrypted value.

As an example, suppose we have a file with the following contens:
```
some_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          65653039303432386134336262653763303264383664383862616330343032653934623465643937
          3766534442233030316266663862346336363937323233320a363438336631383966626237303838
          38653763373663323345623564333931306432373363613336376138323331376466376364656531
          3635393663386335340a564524626433366439323638613538656562366238656463633638616237
          6562
```

when decrypting with `savvy decrypt` this will become:
```
@savvy replace: some_password: my_secret_value
some_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          65653039303432386134336262653763303264383664383862616330343032653934623465643937
          3766534442233030316266663862346336363937323233320a363438336631383966626237303838
          38653763373663323345623564333931306432373363613336376138323331376466376364656531
          3635393663386335340a564524626433366439323638613538656562366238656463633638616237
          6562
```
When (re)encrypting this file with `savvy encrypt`, savvy will check if the new value in the first line is still the same as the old secret value when decrypting the second line. If this is the same, then the old encryption of the second part is being used, so there will be no change for version control.

## Splitting and merging
Instead of decrypting all vault vars and placing them in the original file with a `@savvy replace` marker,
it is also possible to put them in a separate "mergefile".
The secrets in this mergefile can be edited, and then merged back into the original file using the `savvy merge` command.
The flow would be something like
```
savvy split <file>
vi savvy.mergefile
savvy merge <file>
```
The default mergefile is `savvy.mergefile`.
It is also possible to specify a different merge file, using the `-m` or `--mergefile` option.
This option also works for the normal `savvy decrypt` and `savvy encrypt` commands.
In fact `split` and `merge` are basically synonyms for `decrypt` and `encrypt` with a default mergefile.
So the example above is similar to:
```
savvy decrypt -m savvy.mergefile <file>
vi savvy.mergefile
savvy encrypt -m savvy.mergefile <file>
```

## Censoring
Savvy can also censor all by replacing all secrets with some censored value.
This basically works by first reading all secrets just as `savvy view` does.
It also automatically adds the vault password to the secrets to be censored.
Then the standard input is read, and in each line, any of those secrets is replaced
by `@censored<key>`  where the key is the key of that secret.
This way logfiles can be filtered so that no secrets are shown,
but it is clear which secret was used.

The typical way to use this would be to censor all output from a playbook:
```
ansible-playbook ... | savvy censor
```
You may specify a different secrets file, just as in `savvy view`,
but by default `group_vars/all/vars.yml` is used.
This command might be made more flexible in the future,
e.g. by adding extra dictionaries of secrets, or using different input and output.

## Automatic password generation
It is also possible to let savvy generate a password for you using the following syntax:
```
@savvy generate: some_password
```
When running savvy-encrrypt, this will generate a random password and encrypt this.
By default the password will be 16 alphanumeric characters.
Future options will make it possible to specify a different length and character set.

## Future @savvy options
The `@savvy` annotation can support several options.
These options must be placed after the `@savvy` annotation and before the colon, separated by spaces.
Anything after the colon will be the password (with leading and trailing whitespace trimmed).
```
# the password will be encrypted
@savvy: some_password: my_secret_value

# the password will be encrypted in a single-line format
@savvy singleline: some_password: my_secret_value

# the password will be encrypted using an ansible vault-id
@savvy id=dev: some_password: my_secret_value

# A random password of length 20 will be generated abd enrypted (colon is not needed)
@savvy generate length=20: some_password

# Do nothing. This is a reminder that the user must manually fill in this value
@savvy multi-line TODO: api_key: Get the password from the cloud provider

```
## Input, output and merge files
When a user issues a command it can take several arguments about the files.
```
savvy <command> [<inputfile>] [-m <mergefile>] [-o <outputfile>]
```
Furthermore several environment variable can be used:
- SAVVY_INPUTFILE
- SAVVY_MERGEFILE
- SAVVY_OUTPUTFILE

There are several rules trying to be as easy to use, and consequent:
- The inputfile precedence is:
  - command line parameter `<inputfile>`
  - environment option SAVVY_INPUTFILE
  - default value: vars.yml
- There will always be only one file written.
- This file will only be written at the end of the command, to prevent overwriting the file when errors occur.
- The default outputfile for `decrypt`, `encrypt` and `merge` will be the inputfile, thus changing the inputfile
- The default outputfile for `split` it will be the file named savvy.merge.
- The outptufile specified by commandline `-o` will take highest precedence, except for `split` where the `-m` is even higher
- The outputfile precedence is:
  - *command line option `-m` or `--mergefile`: only for `split` command*
  - command line option `-o` or `--outputfile`
  - *environment option SAVVY_MERGEFILE: only for `split` command*
  - environment option SAVVY_OUTPUTFILE: for other commands
  - default value
- For encrypt and merge input is read form a mergefile
  - for `encrypt` the inputfile is used, but only `@savvy` lines are parsed
  - for `merge` a mergefile can be specified:
    * command line option `-m` or `--mergefile`
    * environment option SAVVY_MERGEFILE
    * default value: savvy.mergefile
This is summarized in the table below, with the difference highlighted:

| command | inputfile | merge-input | outputfile |
|---------|-----------|-------------|------------|
| decrypt | file <br/> SAVVY_INPUTFILE <br/> vars.yml | | -o outputfile <br/> SAVVY_OUTPUT <br/> **inputfile** |
| split   | file <br/> SAVVY_INPUTFILE <br/> vars.yml | | **-m mergefile** <br/> -o outputfile <br/> **SAVVY_MERGEFILE** <br/> **savvy.mergefile** |
| encrypt | file <br/> SAVVY_INPUTFILE <br/> vars.yml |  **inputfile** | -o outputfile <br/> SAVVY_OUTPUT <br/> **inputfile** |
| merge | file <br/> SAVVY_INPUTFILE <br/> vars.yml | -m mergefile <br/> SAVVY_MERGEFILE <br/> **savvy.mergefile** | -o outputfile <br/> SAVVY_OUTPUT <br/> **inputfile** |
| view | file <br/> SAVVY_INPUTFILE <br/> vars.yml | | |
