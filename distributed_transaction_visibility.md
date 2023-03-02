
----
[[File:MyPicture_github.jpg]]

[[Koichi]]
 
== Global Visibility for distributed R/W workload amount PostgreSQL database cluster ==

=== Goal of the project ===

* Provide consistent visibility to distributed R/W transactions spanning into more than one postgres database instance.
* No impact to local transactions.

=== Problem to solve ===

We have two-phase commit protocol to maitain write consistency of groups of transactions as a distributed transaction.

In this case, final "commit" timestamp is different in database instances.

Because of this, result of write can be visible in some of involved instances but invisible in others.
This is known as distributed read anomaly.

We need to solve this read anomaly problem.

=== Outline of the solution ===

* We define global transaction identifier unique to each distributed transactions and same among transactions begonging to a same global transaction.
* Use global snapshot for global transactions in row visibility check.

=== Definition and structure of a distrubuted transaction ===

* Any distributed transaction has one "root" transaction which starts the transaction.
* Other transactions in a distributed transaction must be invoked by the "root" transaction or a "child transaction", which is invoked directly or indirectly by the root transaction.
* "'''root'''" transaction is given a global transaction id ('''GXID''') as <code>(dbid, gxid)</code>, where <code>dbid</code> is identifier of the database root transaction is running and gxid is unique number given in ascending order by global transaction proxy as described below.
Typically, database system identifier given by initdb can be used for this purpose.   This depends upon implementation and we can choose any other means as long as dbid is unique.  In the case of database system identifier, this is copied by cloning the database by `pg_basebackup` and streaming replication, for example.   We may need a means to assign different database system identifier if such cloned database should be used as separate database instance.
* When "root" transaction and its "'''child'''" transaction invokes child transactions, such "parent" transactions has to propagate its global transaction Id to their children.   Locally, this global transaction ID has to be associated with the local transction.

=== Assignment of global transaction ID ===

* We need separate facility to manage global transaction information in a given postgreSQL cluster, called "'''global transaction proxy'''" ('''GTP''').
* When a transaction wants to be the root transaction and would like to provide and receive consistent visibility need to connect to GPT and request for new GXID.   GTP assignes gxid and record this with root transaction's dbid as <code>(dbid, gxid)</code> pair.  Please note that this is needed not only for write transaction but for read transaction which need consistent visiblity among different database instances.
* A root transact has to propagate this GXID to its children.   When a child is invoking its child, then it also has to propagate this GXID, which should be associated to the local transaction of the child.
* When a root transaction (and all its children) finishes with final COMMIT, it has to report GTP that the transaction finished.   GTP records this is finished.   Maintenance of GXID information at GTP will be given below.

=== Global snapshot and its handling in local transactions. ===

* The root has to ask for snapshot of running global transactions to GTP.   GTP collects GXID of running global transactions and provides this set of GXIDs as a global snapshot.
* Root has to propagate this global snapshot to its children and so forth.   This is associated with the local transaction of children.
* In visibility check, if a local XID (in xmin/xmax and others) is associated with GXID (we may need a flag in each row to indicate this), such transaction visibility should be determined based on its GXID appearing in the snapshot (note that in global snapshot, both terminated GXID and runing GXID appears, depending upon its syntax).  Details will be given below.
* Visibility of local transactions not associated with GXID can be handled in the same manner as current usual local transactions.
* For a local transaction without GXID, there are no global snapshot and no GXID is used for visibility check even target row's xid is associated with GXID.   Thus, for the local transactions, there will be no additional overhead.

=== Global snapshot data format ===

Global snapshot data can be represented in the following format:

<code>
(GXID_0_0, ..., GXID_0_M) GXMIN (GXID_1_0, ..., GXID_1_N) GXMAX
</code>

<code>GXMIN</code> means global tansaction whose gxid is smaller than this value are all terminated (commited/aborted), unless appearing in the list <code>(GXID_0_0, ... GXID_0_M)</code>.
This is used to save the amount of snapshot especially when only very few number of global transactions are alive for error ercovery (typically local commit failure).

<code>GXMAX</code> means this is the maximum value of currently assigned gxid and global transactions with gxid larger than GXMAX has to be regarded as running.

<code>(GXID_1_0, .... , GXOD_1_N)</code> represents a set of GXIDs whose gxid is between <code>GXMIN</code> and <code>GXMAX</code> and has already terminated.  The reason for this is we cannot guarantee the transaction is running when assocaited database instance is down after the transaction is reported to begin and before it reports to be terminated.

From the above, there are multiple snapshot representation possible for a given situation.  GPT should provide the minumum size snapshot by calculating optimal value for <code>GXMIN</code> and <code>GXMAX</code>.

=== Global snapshot maintenance in GTP ===

GXID in GTP cannot be removed in GTP immediately because we have a chance that other global transaction depends upon this.

When a transaction terminates, its GXID is marked as finished and is associated with set of running GXIDs as referring GXIDs.   Each of such GXID can be removed from the list when it terminates.  When referring GXID becomes zero, then such GXID information can be removed from GTP.

When a database instance gets down and restarts, all the associated transaction are automatically aborted.   In this case, each database instance should report this to GTP and GTP can clean up its global transaction information if dbid matches to such database instance.

Please note that such GXID will not appear in the snapshot requested after it ternates.   This can be used to calculate GXMIN in the snapshot to minimize the amount of global snapshot.

=== Global snapshot maintenance in local database instances ===

Loclal database instances cannot remove GXID associated with local immediately when this GXID terminates because other global transaction can refer to this afterwords.   Local database instance should inquire GTP about valid GXID periodically to prevent bloat.    If such GXID is not present at GTP, local database instance can remove this.   Please note that such remaining GXID-XID mapping does not affect the visibility check correctiveness but can impact the performance.

=== Database instance beloging to more than one database cluster ===

We can allow a database instance can belong to more than one cluster.   In this case, we need to extend the above with cluster-id.   Any global transaction need to determine which cluster it belongs to and GXID and global snapshot request to appropriate GTP.   In this case, GXID can be represented as (cluster-id, dbid, gxid).

