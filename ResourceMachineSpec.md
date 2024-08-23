# Resource Machine Specification

## Notation

Every function and field in the Resource Machine has an associated FiniteField. For now we use a single FiniteField type that wraps a `Nat` to represent them all.

<!---
```juvix "import field"
import Simulator.RM.Field open;
```

```juvix "resource imports" +=
import Stdlib.Prelude open;
<<<import field>>>
```

```juvix "field imports"
import Stdlib.Prelude open;
```


```juvix Simulator/RM/Field.juvix
module Simulator.RM.Field;

<<<field imports>>>

<<<field body>>>
```
--->

```juvix "field body" +=
type FF := mk {
    value : Nat
};

```

## Resource

<!---
```juvix Simulator/RM/Resource.juvix
module Simulator.RM.Resource;

<<<resource imports>>>

<<<resource body>>>
```
--->

The atomic unit of the ARM state is called a resource. Resources are immutable,
they can be created once and consumed once, which indicates that the system
state has been updated. The resources that were created but not consumed yet
make the current state of the system.

A _resource_ is a composite structure.

* l - is a succinct representation of the predicate associated with the
resource called **resource logic**
* label - specifies the fungibility domain for the resource. Resources within
  the same fungibility domain are seen as equivalent kinds of different
  quantities. Resources from different fungibility domains are seen and treated
  as distinct asset kinds. This distinction comes into play in the balance check
  described later.
* q - is a number representing the quantity of the resource
* v - the fungible data associated with the resource. It contains extra
information but does not affect the resource’s fungibility
* eph - is a flag that reflects the resource’s ephemerality. Ephemeral resources
  do not get checked for existence when being consumed
* nonce - guarantees the uniqueness of the resource computable components
* cnk - is a nullifier key commitment. Corresponds to the nullifier key nk used
  to derive the resource nullifier
* rseed - randomness seed used to derive whatever randomness needed

The cnk is the nullifier public key.

The logic function field is the Nat corresponding to the bytes of the hash of the Nock representation of the logic function. Conceptually it is: `hash >> anomaEncode`
  
```juvix "resource body" +=
type Resource := mk {
  l : FF;
  label : FF;
  q : FF;
  v : FF;
  eph : FF;
  nonce : FF;
  cnk : FF;
  rseed : FF
};

```

## Computable Components

### Commitment

<!---

```juvix Simulator/RM/Commitment.juvix
module Simulator.RM.Commitment;
import Stdlib.Prelude open;
import Anoma.Data.ByteArray open;

<<<commitment body>>>
```
--->

For a resource r, _resource commitment_ is computed. The commitment is tied to
the created resource but does noot reveal information about the resource beyond
the fact of creation.

```juvix "commitment body" +=
type Commitment := mk {
  value : ByteArray
};
```

### Commitment implementation

In order to define a commitment of a resource we need a hash implementation:

```juvix "resource imports" +=
import Simulator.Data.Hash open using {hash};
```

We can now define a commitment of a resource in terms of this.

```juvix "resource imports" +=
import Simulator.RM.Commitment as Commitment open using {Commitment};
```

```juvix "resource body" +=
commitment : Resource -> Commitment := hash >> Commitment.mk;
```

The resource commitment is also used as the resoruce's address in the content-addressed storage.

### Commitment Accumulator

All resource commitments are stored in an append-only data structure called a _commitment accumulator_.

We will use a Merkle tree for this.

<!---

```juvix Simulator/RM/CMacc.juvix
module Simulator.RM.CMacc;

<<<cmacc imports>>>

<<<cmacc body>>>
```
--->

```juvix "cmacc imports" +=
import Stdlib.Prelude open;
import Simulator.Data.MerkleTree as MerkleTree open using {MerkleTree; MerkleProof};
import Simulator.RM.Commitment as Commitment open using {Commitment; module Commitment};
import Anoma.Data.ByteArray as ByteArray open using {ByteArray};
```

We wrap the MerkleTree types using the names from the spec:

```juvix "cmacc body" +=
type CMacc := mkCMacc {
  merkleTree : MerkleTree
};

type Witness := mkWitness {
  proof : MerkleProof
};

type CMaccValue := mkCMaccValue {
  root : ByteArray
};

```

The commitment accumulator CMacc must support the following functionality:


* Add (acc, cm) adds an element to the accumulator, returning the witness used to prove membership.

```juvix "cmacc body" +=
add (acc : CMacc) (cm : Commitment) : Pair Witness CMacc :=
  case MerkleTree.insertAndProve (CMacc.merkleTree acc) (Commitment.value cm) of
    | (p, t) := (mkWitness p, mkCMacc t);

```

* Witness(acc,vm) for a given element, returns the witness used to prove membership if the element is present, otherwise returns nothing.

```juvix "cmacc body" +=
witness (acc : CMacc) (cm : Commitment) : Maybe Witness :=
  Commitment.value cm |> MerkleTree.generateProof (CMacc.merkleTree acc) |> map mkWitness; 
  
```

* Verify(cm,w,val) verifies the membership proof for an element cm with a membership witness w for the accumulator value val.

```juvix "cmacc body" +=
verify (cm : Commitment) (w : Witness) (val : CMaccValue) : Bool :=
  MerkleTree.verifyProof (Witness.proof w) (CMaccValue.root val) (Commitment.value cm);
  
```

* Value(acc) returns the accumulator value.

```juvix "cmacc body" +=
value (acc : CMacc) : CMaccValue := CMacc.merkleTree acc |> MerkleTree.rootHash |> mkCMaccValue;
```

### Nullifier

A resource nullifier is a computed field, the publishing of which consumes the associated resource.

A nullifier is computed from the resource's plaintext and a key called a nullifier key.

For this we need to import the PrivateKey type, and anomaSignDetached:

```juvix "resource imports" +=
import Anoma.Types open using {PrivateKey; Signature};
import Anoma.System open using {anomaSignDetached};
import Anoma.Data.ByteArray as ByteArray open using {ByteArray};
```

And define the nullifier by signing the resource plaintext with a key:

```juvix "resource body" +=

type Nullifier := mkNullifier {
  signature : Signature
};

nullifier (privateKey : PrivateKey) (r : Resource) : Nullifier := anomaSignDetached r privateKey |> mkNullifier;

```

Every time a resource is consumed it has to be checked that the resource existed before (the resources commitment is in the CMTree) and has not been consumed yet (is not in the NFset).

<!---

```juvix Simulator/RM/NFset.juvix
module Simulator.RM.NFset;

import Stdlib.Prelude open;
<<<nfset imports>>>

<<<nfset body>>>
```
--->

We define an `NFset` using a `Set` from `juvix-containers`.

For this we'll need an `Ord` instance for `Nullifier`

```juvix "resource imports" +=
import Anoma.Types open using {module Signature};
```


```juvix "resource body" +=
instance
SignatureOrdI : Ord Signature := mkOrd (Ord.cmp on Signature.unSignature);

instance
NullifierOrdI : Ord Nullifier := mkOrd (Ord.cmp on Nullifier.signature); 

```

```juvix "nfset imports" +=
import Data.Set as Set open using {Set};
import Simulator.RM.Resource open using {Nullifier};
```

And define NFset via a newtype:

```juvix "nfset body" +=
type NFset := mk {
  unNFset : Set Nullifier
};

```

The nullifier set supports the following functionality:

* write(nf) adds an element to the nullifier set.

```juvix "nfset body" +=
write (nf : Nullifier) (s : NFset) : NFset := Set.insert nf (s |> NFset.unNFset) |> mk;

```

* exists(nf) checks if the element is presen in the set

```juvix "nfset body" +=
exists (nf : Nullifier) (s : NFset) : Bool := Set.member? nf (s |> NFset.unNFset);

```

### Kind

A kind is computed from the logic and label of a resource.

```juvix "resource body" +=
type Kind := mkKind {
  unKind : ByteArray;
};

kind (r : Resource) : Kind := hash (Resource.l r, Resource.label r) |> mkKind;

```

### Delta

Resource deltas are used to reason about the total quantities of different kinds of resources in transactions.

In general these are need to have cryptographic properties but for the purposes of the simulator we will use a Pair of kind and quantity.

```juvix "resource body" +=
type Delta := mkDelta {
  kind : Kind;
  quantity : FF
};

delta (r : Resource) : Delta := mkDelta (kind r) (Resource.q r);

```

### Resource Logic

A resource logic is a function that approves the creation and consumption of a resource.

<!---

```juvix Simulator/RM/ResourceLogic.juvix
module Simulator.RM.ResourceLogic;

import Stdlib.Prelude open;
<<<resource_logic imports>>>

<<<resource_logic body>>>
```
--->

We will need to import `Resource`, `Nullifier`, `Commitment` to use it.

```juvix "resource_logic imports" +=
import Simulator.RM.Resource as Resoruce open using {Resource; module Resource; Nullifier};
import Simulator.RM.Commitment open;
import Simulator.RM.Field open;
```

We will also need to import `Set`:

```juvix "resource_logic imports" +=
import Data.Set as Set open using {Set};
import Data.Map as Map open using {Map};
import Anoma.Types open using {Signature};
```

Resource logic takes as input a subset of resources created and consumed in the action.

A logic function also takes some set of custom inputs called app_data. This is a Map where the values also have deletion criteria.

```juvix "resource_logic body" +=

type Timestamp := mkTimestamp {
  unTimestamp : Nat
};

type Block := mkBlock {
  unBlock : Nat
};

type SigOverData := mkSigOverData {
  sig : Signature;
  data : Nat
};

type EitherPredicate := mkEitherPredicate {
  predicate1 : DeletionCriteria;
  predicate2 : DeletionCriteria
};

type DeletionCriteria :=
  storeForever 
  | deleteAfterTimestamp Timestamp
  | deleteAfterBlock Block
  | deleteAfter SigOverData
  | deleteAfterPredicate EitherPredicate;

type AppDataValue := mkAppDataValue {
  value : FF;
  deletionCriteria : DeletionCriteria
};

type AppData := mkAppData {
  data : Map FF AppDataValue
};

```

For this we need `FF` to have an ord instance:

```juvix "field body" +=
instance
FieldOrdI : Ord FF := mkOrd (Ord.cmp on FF.value);

```

#### Instance

The tag identifies the resource being checked. How is this used?

```juvix "resource_logic body" +=
type Instance := mkInstance {
  nfs : Set Nullifier;
  cms : Set Commitment;
  tag : FF;
  app_data : AppData;
};

```

#### Witness

These are not referred to in the text of the report, so I'm not sure what they represent. Do custom imports relate the the app_data in some way?

```juvix "resource_logic body" +=
type Witness := mkWitness {
  input : Set Resource;
  output : Set Resource;
  custom : Set Nat
};

```

#### Constraints

The body of the logic function will check that the inputs/outputs are valid.

#### Resource logic signature

I think this means that the resource logic signature will look like:

```juvix "resource_logic body" += 
ResourceLogic : Type := Instance -> Witness -> Bool;
```

## Proving Systems

This relates to proofs of knowledge for a valid transaction. In the simulator we will use a trivial transparent system where the knowledge is proven by publically revealing it.

<!---
```juvix Simulator/RM/ProvingSystem.juvix
module Simulator.RM.ProvingSystem;

import Stdlib.Prelude open;
<<<proving_system imports>>>

<<<proving_system body>>>
```
--->

```juvix "proving_system imports" +=
```

In general a proving system is parametrize by the following:

* Proof
* Instance - the subset of inputs required to both create and verify a proof
* Witness - the subset of inputs used to produce but not verify a proof
* Proving key - The data required, along with the instance and the witness to produce a proof.
* Verifying key - The data required, along with the witness to verify a proof.

```juivx "proving_system body" +=

trait
type ProvingSystem (Proof : Type) (Instance : Type) (Witness : Type) (VerifyingKey : Type) (ProvingKey : Type) := mkProvingSystem {
  prove : ProvingKey -> Instance -> Witness -> Proof;
  verify : Proof -> Instance -> VerifyingKey -> Bool
};

```

We can define a trivial proving system:

```juvix "proving_system body" +=
instance
TrivialPS {A} : ProvingSystem A A Unit Unit Unit := mkProvingSystem@{
  prove (_ : Unit) (a : A) (_ : Unit) : A := a;
  verify (proof : A) (_ : A) (_ : Unit) : Bool := true
};


```



## Action

An action defines a proof context, a proof created in the context of an action is assumed to have guaranteed access only to the resources associated with the action.

<!---
```juvix Simulator/RM/Action.juvix
module Simulator.RM.Action;

import Stdlib.Prelude open;
<<<action imports>>>

<<<action body>>>
```
--->

Let's import the types we need:

```juvix "action imports" +=
import Simulator.RM.Resource open using {Nullifier};
import Simulator.RM.Commitment open using {Commitment};
import Simulator.RM.ResourceLogic open using {AppData};
import Data.Set as Set open using {Set};
```

```juvix "action body" +=
type Action (Proof : Type) := mkAction {
  cms : Set Commitment;
  nfs : Set Nullifier;
  proofs : Set Proof;
  appData : AppData
};

```
