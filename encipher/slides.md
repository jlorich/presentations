## ssh-key based secrets storage in Ruby
####Joey Lorich


## Things I don't like

- Keeping secrets and configuration information in repositories
- Keeping plaintext secrets files (dotenv, application.secrets, figaro)
- Manually editing secrets files on servers
- Passing around secrets files between developers
- No record of who changes the secrets files and when


## Got a problem? Why not build something new

Leverage some common technologies to help solve these issues.


## Public key cryptography

- Widely used method to secure communication over open networks (e.g. the internet)
- One key to encrypt, a different key to decrypt
- Publicly expose encryption key, keep decryption key secret


## SSH

- Conveniently, most of us already have keys we use often


## Encipher

[github.com/jlorich/encipher](https://github.com/jlorich/encipher)

A secure, convenient, commitable, deployable, secrets storage system.

Still in alpha, but functioning

Open for criticism


## What it is

Use public ssh keys to encrypt secrets for each user.  Anyone with a valid private key can decrypt the secrets and re-encrypt them to enable access for others, or add/edit/delete keys with visible public keys.


## What it isn't

Encipher is not meant to ensure a tamper-free secrets database, only to ensure that people can only read secrets they're supposed to.  You should always watch for changes to both gem code and the database itself.


## Gemspec

```
  spec.add_development_dependency "bundler", "~> 1.6"
  spec.add_development_dependency 'rake', '~> 10.3.2'
  spec.add_development_dependency 'pry-byebug', '~> 1.3.3'
  spec.add_development_dependency 'rspec', '~> 3.0.0'
  spec.add_development_dependency 'clint_eastwood', '~> 0.0.1'

  spec.add_dependency 'thor', '~> 0.19.1'
  spec.add_dependency 'highline', '~> 1.6.21'
  spec.add_dependency 'awesome_print', '~> 1.2.0'
  spec.add_dependency 'sqlite3', '~> 1.3.9'
  spec.add_dependency 'data_mapper', '~> 1.2.0'
  spec.add_dependency 'dm-sqlite-adapter', '~> 1.2.0'

  spec.add_dependency 'recursive-open-struct', '~> 0.5.0'
  spec.add_dependency 'deep_merge', '~> 1.0.1'
  
  spec.add_dependency 'net-ssh', '~> 2.9.1'

  spec.add_dependency 'dot_configure', '~> 0.0.1'
  spec.add_dependency 'exedit', '~> 0.0.2'
  
  spec.executables = ['encipher']

  ```


##Five main classes

- Security
- Environment
- Vault
- Encipher
- CLI


## Security
 - Where everything cryptographic happens
 - Use `Net::SSH::KeyFactory` to load our private ssh key (prompts for key password if needed). Returns an `OpenSSL::PKey::RSA`
 - With this we can call `@private_key.public_key.to_pem` to get our PEM formatted string
 - I'm using the PEM string because it's also how I'm storing others public keys
 - Use `OpenSSL::PKey::RSA.new(@public_key).public_encrypt("secret")` to encrypt a secret
 - Use `@private_key.private_decrypt(encrypted_string)` to decrypt the secret


## Vault

 - Interfaces with the defined DataMapper objects
 - Used to locate, lock, unlock, and compare secrets
 - Also manages enrollment/revoking of keys


## Environment

 - A subclass of Vault
 - Implements encryptions of secrets on a per-user basis
 - Provides a simple interface to access secrets


## Encipher
 - The public interface of the gem.  What'd you'd reference in Rails, Sinatra, etc.

```
Encipher.configure do |config|
  config.keypath = '~/.ssh/my_server_key'
  config.env = :staging
end
```

```
Encipher.require(:db_password)
```

```
conn = PG.connect( dbname: 'sales', password: Encipher.secrets[:db_password]) 
```


## CLI

 - How I forsee most developers using the gem


## Examples!


##Potential enhancements

 - Keep the list of public keys encrypted in the same manner as secrets.
 - Keep a hash of the secrets in the DB to alert other users of changes
 - Remote database support - keep secrets in a shared off-site location
 - Automatic loading into Rails.secrets
 - Automatic conversion from dotenv


## Questions?