#Transaction Life Cycle

This document attempts to describe the clixon transaction to a plugin developer.

You should think of clixon as a database and not as a way to describe how to configure a 
system. This means that its job is not to suggest or enforce a method or order for 
configuring a system. 

An important concept is the acronym “ACID”

“Atomicity” guarantees that each transaction is a single operation, which either succeeds 
or fails completely. 

“Consistency” ensures that a transaction can only bring the database from one valid state 
to another. This prevents database corruption by an illegal transaction but does not 
guarantee that a transaction is correct.

“Isolation” ensures that concurrent execution of transactions leaves the database in 
the same state that would have been obtained if the transactions were executed sequentially.

“Durability” guarantees that once a transaction has been committed, it will remain committed. 

The key data structure for passing information from clixon to a plugin is transaction_data_t 
structure:

```
typedef struct {
    uint64_t   td_id;       /* Transaction id */
    void      *td_arg;      /* Callback argument */
    cxobj     *td_src;      /* Source database xml tree */
    cxobj     *td_target;   /* Target database xml tree */
    cxobj    **td_dvec;     /* Delete xml vector */
    size_t     td_dlen;     /* Delete xml vector length */
    cxobj    **td_avec;     /* Add xml vector */
    size_t     td_alen;     /* Add xml vector length */
    cxobj    **td_scvec;    /* Source changed xml vector */
    cxobj    **td_tcvec;    /* Target changed xml vector */
    size_t     td_clen;     /* Changed xml vector length */
} transaction_data_t;
```

| field | description |
| ----: | :---- |
| td_id	| transaction ID, can be used to group a series of callbacks into a transaction |
| td_arg | ??? |
| td_src | pointer to complete source database XML. |
|  |  while this is typically the same thing as the running database, you should not assume this in your code. |
| td_target | pointer to complete target database XML |
| | while this is typically the same thing as the candidate database, you should not assume this in your code. |
| td_dvec | pointer to a vector containing pointers to nodes that have been deleted in the source tree |
| td_dlen | length of the vector in td_dvec |
| td_avec | pointer to a vector containing pointer to nodes that have been added in the target tree |
| td_alen | vector length of td_avec |
| td_scvec | pointer to the original nodes in the source database that have been changed in the target database. |
| td_tcvec | pointer to the new nodes in the target database that have been changed from the source database |
| td_clen | vector length of the change vectors |

Note that there is a correspondence between the two change vectors. Index 0 in the source vector points at the original value, and index 0 in the target vector point to the new value. 

## begin callback

The begin callback indicates that a transaction is starting, but that the system level validation
has not begun yet.

Not much has happened at this point, and you should not rely on the transaction_data [?]

## validate callback

The validate callback is triggered by a client requesting a validation of the current change set.
Before this callback, clixon will have performed the system level validation, which primarily is 
to validate the YANG. It is recommended that you place as much of the validation in to the YANG 
spec as possible. This does two things, primarily it allows the user of the YANG model to 
understand the constraints clearly, and secondarily, it is more efficient in that the plugin 
calls do not happen if it cannot pass the system level validation.

YANG has a limited set of constraints available and you want to use them as fully as possible. 
Some logic cannot be implemented in YANG. This is an ideal candidate to put into the plugin
callback.

This callback must only validate the transaction data and return a success/failure indication. 
It must not change the state of the system in any way.

Ideally, successful completion guarantees that the commit will be without error. See the commit 
information below on why this is important.

If a failure status is returned to clixon then the transaction is cancelled, and the transaction 
abort callback will be called from clixon.

Note that unlike the other callbacks the validate callback can be called multiple times in a 
transaction. There are three methods for a client to access the model, NETCONF, RESTCONF, and 
the CLI. Clixons model is based on the NETCONF as it is the common base for internal operation 
of clixon and the RESTCONF and CLI models can be expressed in terms of the NETCONF model. 
RESTCONF completes an entire transaction in a single request. CLI is dependent upon the 
developer of the cli specifications – you can use either model or a hybrid.

## complete callback

this indicates to the plugin that the validation was successful

transaction data at this point will be valid.

## commit callback

called when the client requests a commit.

this callback receives valid transaction data.

this callback should perform the actions detailed in the transaction data.

Ideally all possible errors should be detected in the validate callback, but this clearly 
is a dream. If an error occurs in the commit phase, then the transaction abort callback will 
be called. It is important that if an error occurs that the actions that completed successfully 
must be reverted to maintain atomicity and consistency.

## end callback

called when a commit has successfully completed.

you should not depend upon transaction data being valid at this point.

## abort callback

the abort callback is used to notify the plugin that the transaction has failed.

if a validation has failed, then there is not much to do – the changeset is tossed and no 
plugin action is required.

if a commit has failed, then things get interesting. this callback receives transaction 
where the target and source databases are swapped, giving the plugin a way back to the 
original configuration. the abort routine should unwind the commit as best possible.

