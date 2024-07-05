# Juvix Anoma Test

A library for testing Anoma applications.

## API

The API for the library is contained in [`Test.Anoma`](./Test/Anoma.juvix).

## Examples

An example of verifying a transaction using the library can be found in [`tests/AnomaExample.juvix`](./tests/AnomaExample.juvix).

```
$ juvix eval tests/AnomaExample.juvix
[INFO] Verifying transaction
[INFO] Verifying resource set
[INFO] ... Resource set valid
[INFO] Verifying delta sum
[INFO] Verifying logic functions
[INFO] ... All logic functions passed
```

An example of integration with `juvix-test` can be found in [`tests/Main.juvix`](./tests/Main.juvix).

```
$ juvix eval tests/Main.juvix
Test suite 'Anoma tests'
Valid transaction is valid		OK
Transaction with invalid logic function is invalid		OK
Transaction with empty proofs is invalid		OK
Transaction with invalid public key is invalid		OK
All tests from test suite 'Anoma tests' complete
Suite passed
```
