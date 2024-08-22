# Resource Machine Specification

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

```juvix Simulator/RM/Resource.juvix
module Simulator.RM.Resource;

<<<resource imports>>>

<<<resource body>>>
```

```juvix Simulator/RM/Field.juvix
module Simulator.RM.Field;

<<<field imports>>>

<<<field body>>>
```


```juvix Simulator/RM/Commitment.juvix
module Simulator.RM.Commitment;
import Stdlib.Prelude open;
import Simulator.Data.ByteArray open;

<<<commitment body>>>
```

--->

## Notation

Every function and field in the Resource Machine has an associated FiniteField. For now we use a single FiniteField type that wraps a `Nat` to represent them all.

```juvix "field body" +=
type FF := mk {
    value : Nat
}
```

## Resource

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
import Simulator.Data.ByteArray as ByteArray open using {ByteArray};
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
import Simulator.Data.ByteArray as ByteArray open using {ByteArray};
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

