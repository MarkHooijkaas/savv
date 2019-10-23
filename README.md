# savvy-tools
Single Ansible Vault Variable encrYpt/decrYpt Tools

This set of tools is meant to use single variables in a normal unencryted vars file that are encrypted with ansible-vault.
Ansible playbooks support the use of these variables, but the tooling to encrypt and decrypt these vars can be better.

The main planned tools are:
* `savvy encrypt`: encrypt all `@savvy` annotated variables in a file
* `savvy decrypt`: decrypt all `!vault` variables in a file
* `savvy edit`: decrypt, edit, and re-encrypt 

## Goals
* work well with version control system like git
* be able to edit a file with several secret values
* support single and multi line vault formats
* automatically generate new passwords

## Encrypting
To encrypt some variables in a file, these can be marked with a @savvy annotation as follows:
```
some_password: @savvy: BadPassword123
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
some_password: @savvy multiline: my_secret_value
some_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          65653039303432386134336262653763303264383664383862616330343032653934623465643937
          3766534442233030316266663862346336363937323233320a363438336631383966626237303838
          38653763373663323345623564333931306432373363613336376138323331376466376364656531
          3635393663386335340a564524626433366439323638613538656562366238656463633638616237
          6562
```
When (re)encrypting this file with `savvy encrypt`, savvy will check if the new value in the first line is still the same as the old secret value when decrypting the second line. If this is the same, then the old encryption of the second part is being used, so there will be no change for version control.

## Automatic password generation
It is also possible to let @savvy generate a password for you using the following syntax:
```
some_password: @savvy generate
```
When running savvy-encrrypt, this will generate a random password and encrypt this.
By default the password will be 16 alphanumeric characters.
Future options will make it possible to specify a different length and character set.

## Other @savvy options
The `@savvy` annotation can support several options. 
These options must be placed after the `@savvy` annotation and before the colon, separated by spaces.
Anything after the colon will be the password (with leading and trailing whitespace trimmed).
```
# the password will be enrypted
some_password: @savvy: my_secret_value 

# the password will be enrypted in a multi-line format
some_password: @savvy multiline: my_secret_value

# the password will be enrypted using an ansible vault-id
some_password: @savvy id=dev: my_secret_value

# A random password of length 20 will be generated abd enrypted (colon is not needed)
some_password: @savvy generate length=20

# Do nothing. This is a reminder that the user must manually fill in this value 
api_key: @savvy multi-line TODO: Get the password from the cloud provider

```
