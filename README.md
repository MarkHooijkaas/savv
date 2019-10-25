# savvy-tools
Single Ansible Vault Variable encrYpt/decrYpt Tools

This set of tools is meant to use single variables in a normal unencryted vars file that are encrypted with ansible-vault.
Ansible playbooks support the use of these variables, but the tooling to encrypt and decrypt these vars can be better.

The tools are:
* `savvy decrypt`: decrypt all `!vault` variables in a file with special @savvy marker
* `savvy split` :  decrypt all `!vault` variables in a separate mergefile
* `savvy encrypt`: encrypt all `@savvy` annotated variables in a file
* `savvy merge`: encrypt all variables from a mergefile into the main file (not implemented yet)
* `savvy view`: show all encrypted variables in a file
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
 @savvy:some_password: my_secret_value
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
Instead of decrypting all vault vars and placing them in the original file with a savvy marker,
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
In fact `split` and `merge` are basically synonyms for `decode` and `encode` with a default mergefile.
So the example above is similar to:
```
savvy decode -m savvy.mergefile <file>
vi savvy.mergefile
savvy encode -m savvy.mergefile <file>
```

## Automatic password generation
It is also possible to let savvy generate a password for you using the following syntax:
```
@savvy generate: some_password
```
When running savvy-encrrypt, this will generate a random password and encrypt this.
By default the password will be 16 alphanumeric characters.
Future options will make it possible to specify a different length and character set.

## Other @savvy options
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
