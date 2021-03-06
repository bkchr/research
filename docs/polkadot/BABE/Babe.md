====================================================================

Author: Handan Kilinc Alper

Last updated: 26.02.2020

Email: handan@web3.foundation

====================================================================
\(
   \def\skvrf{\mathsf{sk}^v}
   \def\pkvrf{\mathsf{pk}^v}
   \def\sksgn{\mathsf{sk}^s}
   \def\pksgn{\mathsf{pk}^s}
   \def\skac{\mathsf{sk}^a}
   \def\pkac{\mathsf{pk}^a} 
   \def\D{\Delta}
   \def\A{\mathcal{A}}
   \def\vrf{\mathsf{VRF}}
   \def\sgn{\mathsf{Sign}}
\)

# BABE


## 1. Overview

In Polkadot, we produce relay chain blocks using our
 **B**lind **A**ssignment for **B**lockchain **E**xtension protocol,
 abbreviated BABE.
BABE assigns blocks production slots, according to stake,
 using roughly the randomness cycle from Ouroboros Praos [2].

In brief, all block producers have a verifiable random function (VRF)
keys which they register with locked stake.  These VRFs produce secret
randomness which determines when they produce blocks.  A priori, there
is a risk that block producers could grind through VRF keys to bias
results, so VRF inputs must include public randomness created only
after the VRF key.  We therefore have epochs in which we create fresh
public on-chain randomness by hashing together all the VRF outputs
revealed in block creation during the epoch.  In this way, we cycle
between private but verifiable randomness and collaborative public
randomness.

... TODO ...

In Ouroboros [1] and Ouroboros Praos [2], the best chain (valid chain) is the longest chain. In Ouroboros Genesis, the best chain can be the longest chain or the chain which is forked long enough and denser than the other chains in some interval.  We have a different approach for the best chain selection based on GRANDPA and longest chain.  In addition, we do not assume that all parties can access the current slot number which is more realistic assumption.

## 2. BABE 

In BABE, we have sequential non-overlaping epochs $(e_1, e_2,\ldots)$, each of which consists of a number of sequential block production slots ($e_i = \{sl^i_{1}, sl^i_{2},\ldots,sl^i_{t}\}$) up to some bound $t$.  At the beginning of an epoch, we randomly assign each block production slot to a "slot leader", often one party or no party, but sometimes more than one party.  These assignments are initially secrets known only to the assigned slot leader themselves, but eventually they publicly claim their slots when they produce a new block in one.

Each party $P_j$ has as *session key* containing at least two types of secret/public key pair:

* a verifiable random function (VRF) key $(\skvrf_{j}, \pkvrf_{j})$, and
* a signing key for blocks $(\sksgn_j,\pksgn_j)$, possibly the same as the VRF key. 

We favor VRF keys being relatively long lived because new VRF keys cannot be used until well after creation and submission to the chain.  Yet, parties should update their associated signing keys from time to time to provide forward security against attackers who might exploit from creating slashable equivocations.  There are more details about session key available [here](https://github.com/w3f/research/tree/master/docs/polkadot/keys).

Each party $P_j$ keeps a local set of blockchains $\mathbb{C}_j =\{C_1, C_2,..., C_l\}$.  All these chains have some common blocks, at least the genesis block, up until some height.

We assume that each party has a local buffer that contains the transactions to be added to blocks. All transactions in a block is validated with a transaction validation function.


### BABE with GRANDPA Validators $\approx$ Ouroboros Praos

BABE is almost the same as Ouroboros Praos [2] except chain selection rule and the slot time adjustment.


Given that the weight (i.e., stake) of a validator $V_i$ is $w_i$ and the total weigh is $W = w_1 + w_2 + ... + w_n$ where $n$ is the number of validators, the parameter $\theta_i = \frac{w_i}{W}$ and the probability of a validator $V_i$ selected is

$$p = \phi_c(\theta_i) = 1-(1-c)^{\theta_i}$$

where $c$ is a constant. 

In BABE, all validators have same amount of stake so their probability of being selected as slot leaders is equal. Therefore, $\theta_1 = \theta_2 = ... = \theta_n = \theta = \frac{1}{n}$.

The threshold used in BABE for each validator $P_i$ is 

$$\tau = 2^{\ell_{vrf}}\phi_c(\theta)$$

where $\ell_{vrf}$ is the length of the VRF's first output (randomness value).

BABE consists of three phases:

#### 1. Genesis Phase

In this phase, we manually produce the unique genesis block.

The genesis block contain a random number $r_1$ for use during the first epoch for slot leader assignments. Session public keys of initial validators are ($\pkvrf_{1}, \pkvrf_{2},..., \pkvrf_{n}$), $(\pksgn_{1}, \pksgn_{2},..., \pksgn_{n}$).

We might reasonably set $r_1 = 0$ for the initial chain randomness, by assuming honesty of all validators listed in the genesis block.  We could use public random number from the Tor network instead however.

TODO: In the delay variant, there is an implicit commit and reveal phase provided some suffix of our genesis epoch consists of *every* validator producing a block and *all* produced blocks being included on-chain, which one could achieve by adjusting paramaters.

#### 2. Normal Phase

We assume that each validator divided their timeline in slots after receiving the genesis block. They determine the current slot number according to their timeline. If a new validator joins to BABE after the genesis block, this validator divides his timeline into slots with the Median algorithm we give in Section 4.

In normal operation, each slot leader should produce and publish a block.  All other nodes attempt to update their chain by extending with new valid blocks they observe.

We suppose each validator $P_j$ has a set of chains $\mathbb{C}_j$ in the current slot $sl_k$ in the epoch $e_m$.  We have a best chain $C$ selected in $sl_{k-1}$ by our selection scheme, and the length of $C$ is $\ell\text{-}1$. 

Each validator $P_j$ produces a block if he is the slot leader of $sl_k$.  If the first output ($d$) of the following VRF is less than the threshold $\tau$ then he is the slot leader.

$$\vrf_{\skvrf_{j}}(r_m||sl_{k}) \rightarrow (d, \pi)$$

If $P_j$ is the slot leader, $P_j$ generates a block to be added on $C$ in slot $sl_k$. The block $B_\ell$ should contain the slot number $sl_{k}$, the hash of the previous block $H_{\ell\text{-}1}$, the VRF output  $d, \pi$, transactions $tx$, and the signature $\sigma = \sgn_{\sksgn_j}(sl_{k}||H_{\ell\text{-}1}||d||pi||tx))$. $P_i$ updates $C$ with the new block and sends $B_\ell$.


![ss](https://i.imgur.com/Yb0LTJN.png =250x )



In any case (being a slot leader or not being a slot leader), when $P_j$ receives a block $B = (sl, H, d', \pi', tx', \sigma')$ produced by a validator $P_t$, it validates the block  with $\mathsf{Validate}(B)$. $\mathsf{Validate}(B)$ should check the followings in order to validate the block:

* if $\mathsf{Verify}_{\pksgn_t}(\sigma')\rightarrow \mathsf{valid}$ (signature verification),

* if the party is the slot leader: $\mathsf{Verify}_{\pkvrf_t}(\pi', r_m||sl) \rightarrow \mathsf{valid}$ and $d' < \tau_t$ (verification with the VRF's verification algorithm). 

* if $P_t$ did not produce another block for another chain in slot $sl$ (no double signature),

* if there exists a chain $C'$ with the header $H$,

* if the transactions in $B$ are valid.

If the validation process goes well, $P_j$ adds $B$ to $C'$. Otherwise, it ignores the block.


At the end of the slot, $P_j$ decides the best chain with the chain selection rule we give in Section 3.




#### 3. Epoch Update

Before starting a new epoch $e_m$, there are certain things to be completed in the current epoch $e_{m-1}$.
* Validators update
* (Session keys)
* Epoch randomness

If there is a validator update in BABE, this update has to be done until the end of the last block of the current epoch $e_{m-1}$ so that they are able to actively participate the block production in epoch $e_{m+2}$. So, any validator update will valid in the BABE after at least two epoch's later.

The new randomness for the new epoch is computed as in Ouroboros Praos [2]: Concatenate all the VRF outputs of blocks in the current epoch $e_{m-1}$ (let us assume  the concatenation is $\rho$). Then the randomness in epoch $e_{m+1}$:

$$r_{m+1} = H(r_{m-1}||m+1||\rho)$$

This also can be combined with VDF output to prevent little bias by the adversaries for better security bounds. BABE is secure without VDF but if we combine VDF with the randomness produced by blocks, we have better parachain allocation.

## 3. Best Chain Selection

Given a chain set $\mathbb{C}_j$ an the parties current local chain $C_{loc}$, the best chain algorithm eliminates all chains which do not include the finalized block $B$ by GRANDPA. Let's denote the remaining chains by the set $\mathbb{C}'_j$. If we do not have a finalized block by GRANDPA, then we use the probabilistic finality in the best chain selection algorithm (the probabilistically finalized block is the block which is $k$ block before than the last block of $C_{loc}$). 


We do not use the chain selection rule as in Ouroboros Genesis [3] because this rule is useful for parties who become online after a period of time and do not have any  information related to current valid chain (for parties always online the Genesis rule and Praos is indistinguishable with a negligible probability). Thanks to Grandpa finality, the new comers have a reference point to build their chain so we do not need the Genesis rule.


## 4. Clock Adjustment

It is important for parties to know the current slot  for the security and completeness of BABE. For this, validators can use their computer clocks which is adjusted by the Network Time Protocol. However, in this case, we need to trust servers of NTP. If an attack happens to one of these servers than we cannot claim anymore that BABE is secure. Therefore, we show how a party realizes the notion of slots without using NTP. Here, we assume we have a partial synchronous channel meaning that any message sent by a validator arrives at most $\D$-slots later. $\D$ is an unknown parameter.


Each party has a local clock and this clock is not updated by any extarnal source such as NTP or GPS. When a validator receives the genesis block, it stores the arrival time as $t_0$ as a reference point of the beginning of the first slot. We are aware of the beginning of the first slot is not same for everyone. We assume that the maximum difference of start time of the first slot between validators is at most $\delta$. Then each party divides their timeline in slots and update periodically its local clock with the following algorithm. 



**Median Algorithm:** 
The median algorithm is run by all validators in the end of sync-epochs (we note that epoch and sync-epoch are not related). The first sync-epoch $\varepsilon_1$ starts just after the genesis block is released. The other sync-epochs  $\varepsilon_i$ start when the slot number of the last (probabilistically) finalized block is $\bar{sl}_{\epsilon}$ which is the smallest slot number such that  $\bar{sl}_{\varepsilon} - \bar{sl}_{\varepsilon-1} \geq s_{cq}$ where $\bar{sl}_{\varepsilon-1}$ is the slot number of the last (probabilistically) finalized block in the sync-epoch $\varepsilon-1$. Here, $s_{cq}$ is the parameter of the chain quality (CQ) property. If the previous epoch is the first epoch then $sl_{e-1} = 0$. We define the last (probabilistically) finalized block as follows: Retrieve the best blockchain according to the best chain selection rule, prun the last $k$ blocks of the best chain, then the last (probabilistically) finalized block will be the last block of the prunned best chain. Here, $k$ is defined according to the common prefix property. 

The details of the protocol is the following: Each validator stores the arrival time $t_i$ of valid blocks constantly according to its local clock.  In the end of a sync-epoch, each validator retrieves the arrival times of valid and finalized blocks which has a slot number $sl'_x$ where 
* $\bar{sl}_{\varepsilon-1} < sl_x \leq \bar{sl}_{\varepsilon}$ if $\varepsilon > 1$.
* $\bar{sl}_{\varepsilon-1} \leq sl_x \leq \bar{sl}_{\varepsilon}$ if $\varepsilon = 1$.

Let's assume that there are $n$  such blocks that belong to the current sync-epoch and let us  denote the stored arrival times of blocks in the current sync-epoch by $t_1,t_2,...,t_n$ whose slot numbers are $sl'_1,sl'_2,...,sl'_n$, respectively. A validator selects a slot number $sl > sl_e$ and runs the median algorithm which works as follows:


```
for i = 0 to n:
    a_i = sl - sl'_i
    store t_i + a_i * T to lst
lst = sort (lst)
return median(lst)
```

In the end, the validator adjusts its clock by mapping $sl$ to output of the median algorithm.


The following image with chains explains the algorithm with an example in the first epoch where $s_{cq} = 9$ and $k=1$:

![](https://i.imgur.com/jpiuQaM.png)


**Lemma 1:** (The difference between outputs of median algorithms of validators) Asuming that $\delta$ is the maximum network delay, the maximum difference between start time is at most $\delta$.

**Proof Sketch:** Since all validators run the median algorithm with the arrival time of the same blocks, the difference between the output of the median algorithm of each validator differs at most $\delta$. 

**Lemma 2:** (Adjustment Value) Assuming that the maximum total drift on clocks between sync-epochs is at most $\Sigma$ and $2\delta + |\Sigma| \leq \theta$, the maximum difference between the new start time of a slot $sl$ and the old start time of $sl$ is at most $\theta$.

This lemma says that the block production may stop at most $\theta$ at the beginning of the new synch-epoch. 

**Proof Sketch:** With the chain quality property, we can guarantee that more than half of arrival times of the blocks used in the median algorithm sent on time. Therefore, the output of all validators' median algorithm is the one which is sent on time. The details of the proof is in Theorem 1 our paper [Consensus on Clocks](https://eprint.iacr.org/2019/1348). 

Having $\theta$ small enough is important not to slow down the block production mechanism a while after a sync-epoch. For example, (a very extreme example)  we do not want to end up with a new clock that says that we are in the year 2001 even if we are in 2019. In this case, honest validators may wait 18 years to execute an action that is supposed to be done in 2019. 

## 5. Security Analysis

(If you are interested in parameter selection and practical results based on the security analysis, you can directly go to the next section)
BABE is the same as Ouroboros Praos except the chain selection rule and clock adjustment. Therefore, the security analysis is similar to Ouroboros Praos with few changes.


### Definitions
We give the definitions of  security properties before jumping to proofs.

**Definition 1 (Chain Growth (CG)) [1,2]:** Chain growth with parameters $\tau \in (0,1]$ and $s \in \mathbb{N}$ ensures that if the best chain owned by an honest party at the onset of some slot $sl_u$ is $C_u$, and the best chain owned by an honest party at the onset of slot $sl_v \geq sl_u+s$ is $C_v$, then the difference between the length of $C_v$ and $C_u$ is greater or equal than/to $\tau s$.

The honest chain growth (HCG) property is weaker version of CG which is the same definition with the restriction that $sl_v$  and $sl_u$ are assigned to honest validators. The parameters of HCG are $\tau_{hcg}$ and $s_{hcg}$ instead of $\tau$ and $s_{cg}$ in the CG definition.

**Definition 2 (Existantial Chain Quality (ECQ)) [1,2]:** Consider a chain $C$ possessed by an honest party at the onset of a slot $sl$. Let $sl_1$ and $sl_2$ be two previous slots for which $sl_1 + s_{ecq} \leq sl_2 \leq sl$. Then $C[sl_1 : sl_2]$ contains at least one block generated by an honest party.

**Definition 2 (Chain Quality (CQ)) [1,2]:** Chain quality with parameters $\mu \in (0,1]$ and $k \in \mathbb{N}$ ensures that the ratio of honest blocks in any $k$ length portion of an honest chain is $\mu$.

The honest chain quality (HCQ) property is the weaker version of the CQ property. We consider any $k$ proportion of an honest chain $C$ which is generated between two honest slots $sl_1$ and $sl_2$ where $sl_1 \leq sl_1 + s_{hcq}$. The HCQ property says that $C[sl_1 + 1 : sl_2]$ must contain at least $\mu_{hcq}s_{hcq}$ honestly generated blocks.


**Definition 3 (Common Prefix)** Common prefix with parameters $k \in \mathbb{N}$ ensures that any chains $C_1, C_2$ possessed by two honest parties at the onset of the slots $sl_1 < sl_2$ are such satisfies $C_1^{\ulcorner k} \leq C_2$ where  $C_1^{\ulcorner k}$ denotes the chain obtained by removing the last $k'$ blocks from $C_1$, and $\leq$ denotes the prefix relation.



With using these properties, we show that BABE has the persistance and liveness properties. **Persistence** ensures that, if a transaction is seen in a block deep enough in the chain, it will stay there and **liveness** ensures that if a transaction is given as input to all honest players, it will eventually be inserted in a block, deep enough in the chain, of an honest player.

### Security Proof of BABE
We analyze BABE with the NTP protocol and with the Median algorithm. 

We first prove that BABE (both versions) satisfies chain growth, chain quality and common prefix properties in one epoch. Second, we prove that BABE is secure by showing that BABE satisfies persistence and liveness in multiple epochs. 

In Polkadot, all validators have equal stake (the same chance to be selected as slot leader), so the relative stake is $\alpha_i = 1/n$ for each validator where $n$ is the total number of validators. 

We use notation $p_h$ (resp. $p_m$) to show the probability of an honest validator (resp. a malicious validator) is selected. Similarly, we use $p_H$ (resp. $p_M$) to show the probability of *only* an honest validator (resp. malicious validator) is selected. $p_{\bot}$ is the probability of having an emty slot (no validator selected).

$$p_\bot=\mathsf{Pr}[sl = \bot] = \prod_{i\in \mathcal{P}}1-\phi(\alpha_i) = \prod_{i \in \mathcal{P}} (1-c)^{\alpha_i} = 1-c$$

$$p_M = \prod_{i \in \mathcal{P_h}} 1- \phi(1/n) \sum_{i \in \mathcal{P}_m} \binom{\alpha n}{i}\phi(1/n)^i (1- \phi(1/n))^{\alpha n - i} $$

$$p_h = c - p_M$$

$$p_H = \prod_{i \in \mathcal{P_m}} 1- \phi(1/n) \sum_{i \in \mathcal{P}_h} \binom{\alpha n}{i}\phi(1/n)^i (1- \phi(1/n))^{\alpha n - i}$$

$$p_m = c - p_H$$



The validators in BABE with NTP are perfectly synchronized (i.e., the differrence between their clocks are 0). On the other hand, the validators in BABE with the median algorithm have their clocks differ at most $\D + |2\Sigma|$. In BABE with the NTP, any honest validator builds on top of an honest block generated in slot $sl$ for sure if $T> \D$. In BABE with the median algorithm, the honest validadors build on top of an honest block if $\D+ 2|\Sigma| < T - \D$.

**Theorem 1:** BABE with NTP satisfies HCG property with parameters $\tau_{hcg} = p_h(1-\omega)$ where $0 < \omega < 1$ and $s_{hcg} > 0$ in $s_{hcg}$ slots  with probability $1-\exp(-\frac{ p_h s_{hcg} \omega^2}{2})$.

**Proof:** We need to count the honest slots (i.e., the slot assigned to at least one honest validator) (Def. Appendix E.5. in [[Genesis](https://eprint.iacr.org/2018/378.pdf)]) to show the HCG property. The best chain grows one block in honest slots. If honest slots out of $s_{hcg}$ slot are less than $s_{hcg}\tau_{hcg}$, the HCG property is violated. The probability of having a honest slot is $p_h$.

We find below the probability of less than $\tau_{hcg} s_{hcg}$ slots are honest slots. From Chernoff bound we know that

$$\Pr[\sum honest \leq  (1-\omega) p_h s_{hcg}] \leq \exp(-\frac{p_h s_{hcg} \omega^2}{2})$$

$$\tag*{$\blacksquare$}$$







**Theorem 2 (Honest Chain Quality):** Let $\D \in \mathbb{N}$ and let $\tau_{hcg}-\tau_{hcg}\mu_{hcq} > p_m (1+\gamma)$ where $\gamma > 0$, the adversary violates the honest chain quality property in $R$ slots with the probability at most $\exp(\frac{\gamma^2s_{hcq}p_m}{2+\gamma})$.

**Proof:** Assume that we have the best chain $C$ by an honest validator. From honest CG, we know that $|C[sl_i:sl_j]| \geq \tau_{hcg} s_{hcq}$ for all $sl_j - sl_i \geq s_{hcq}$ where $sl_i$ and $sl_j$ assigned to some honest validators. Let us denote that the number of malicous slots between $sl_i$ and $sl_j$ is $m$. We need to show that we have $\mu_{hcq}\tau_{hcg}s_{hcq}$ honest blocks in $C[sl_i:sl_j]$.  The HCQ property is broken if $\tau_{hcg} s_{hcq} - m < s_{hcq} \tau_{hcg}\mu_{hcq}$ where $\tau_{hcg} s_{hcq} - m$ is the number of honest blocks in $C[sl_i:sl_j]$.

$$\Pr[m > s_{hcg}(\tau_{hcg}-\tau_{hcg}\mu_{hcq})> (1+\gamma) p_m s_{hcq}] \leq \exp(-\frac{\gamma^2s_{hcq}p_m}{2+\gamma})$$

$$\tag*{$\blacksquare$}$$

We note that we need the honest chain quality property only for the BABE with the median algorithm.


**Theorem 3 (Existential Chain Quality):** Let $\D \in \mathbb{N}$ and let $\frac{p_h}{c} > \frac{1}{2}$. Then, the probability of an adversary $\A$ violate the ECQ property with parammeters $k_{cq}$ with probability at most $e^{-\Omega(k_{cq})}$.

**Proof (sketch):** If $k$ proportion of a chain does not include any honest blocks, it means that the bad slots are less than the good slots between the slots that spans these $k$ blocks. Since probability of having a  good slots is greater than $\frac{1}{2}$, having more bad slots falls exponentially with $k_{cq}$. Therefore, the ECQ property  is broken  in $R$ slots at most with the probability $e^{-\Omega(k_{cq})}$.

$$\tag*{$\blacksquare$}$$


**Theorem 4 (Common Prefix):** Let $k,\D \in \mathbb{N}$ and let $\frac{p_H}{c} > \frac{1}{2}$, the adversary violates the common prefix property with parammeter $k$ in $R$ slots with probability at most $\exp(− \Omega(k))$  in BABE with NTP.
We should have the condition $\frac{p_H(1-c)^{3\D}}{c} > \frac{1}{2}$ for BABE.



#### Overall Results:

According to Lemma 10 in [[Genesis](https://eprint.iacr.org/2018/378.pdf)]) **chain growth** is satisfied with 
$$s_{cg} = 2 s_{ecq} + s_{hcg} \text{ and } \tau = \tau_{hcg} \frac{s_{hcg}}{2 s_{ecq} + s_{hcg}}$$ and **chain quality** is satisfied with $$s_{cq} = 2 s_{ecq} + s_{hcq} \text{ and } \mu = \tau_{hcq}\frac{s_{hcq}}{2s_{ecq}+s_{hcq}}$$ 


**Theorem 6 (Persistence and Liveness BABE with NTP):** Given that  $k_{cq}$ is the ECQ paramter, $k > 2k_{cq}$ is the CP parameter, $s_{hcg} = k/\tau_{hcg}$, $s_{ecq} = kcq/t$, the epoch length is $R = 2s_{ecq} + s_{hcg}$ BABE with NTP is persistent and live.



**Proof (Sketch):** The overall result says that $\tau = \tau_{hcg}\frac{s_{hcg}}{2s_{ecq}+s_{hcg}} = \frac{k}{s_{hcg}}\frac{s_{hcg}}{2s_{ecq}+s_{hcg}} = \frac{k}{R}$. The best chain at the end of an epoch grows at least $k$ blocks in one epoch thanks to the chain growth.

This implies that at least one of the $k$ blocks is generated by an honest validator thanks to the chain growth. Since $k > 2k_{cq}$, the last $k_{cq}$ block of includes at least one honest block. Therefore, the randomness includes one honest randomness and the adversary can have at most $s_{ecq}$ slots to change the randomness. This grinding effect can be crudely upper-bounded by $s_{ecq}(1-\alpha)nq$.
The randomness generated by an epoch is finalized at latest one epoch later thanks to the common prefix property. Similary, the session key update which is going to be used in three epochs later is finalized one epoch later before a randomness of the epoch where the new key are going to be used starts to leak.
Therefore, BABE with NTP is persistent and live.

$$\tag*{$\blacksquare$}$$

**Theorem 7 (Persistence and Liveness BABE with the Median Algorithm):** Given that $\tau_{hcg}-\tau_{hcg}\mu_{hcq} > p_m (1+\gamma)$ where $\tau_{hcg} = p_h (1-\omega)$, $\mu_{hcq} > 0.5$, the probability that having delay more than $2\D + \Sigma$ is $p_{delay} = \exp(-\frac{ph s_{hcq} \gamma^2}{2+\gamma})$.

We note that we do not need to have $p_{delay}$ negligible becuase it is not related to security.



**These results are valid assuming that the signature scheme with account key is  EUF-CMA (Existentially Unforgible Chosen Message Attack) secure, the signature scheme with the session key is forward secure, and VRF realizing is realizing the functionality defined in [2].**


## 6. Practical Results

In this section, we find parameters of two versions of BABE to achieve the security in BABE.

We fix the life time of the protocol as $\mathcal{L}=3 \text{ years}  = 94670777$ seconds. We denote the slot time by $T$ (e.g., $T = 6$ seconds).
The life time of the protocol in terms of slots is $L = \frac{\mathcal{L}}{T}$. The maximum network delay is $\D$. 


### BABE with the NTP

* Define $\D$
* Choose slot time $T >\D$.
* Decide the parameter $c$ such that the condition $\frac{p_H}{c} > \frac{1}{2}$ is satisfied. If there is not any such $c$, then consider to increase $\alpha$ (honest validator assumption).
* Set up a security bound $p_{attack}$ to define the probability of an adversary to break BABE in e.g., 3 years. Of course, very low $p$ is better for the security of BABE but on the other hand it may cause to have very long epochs and long probabilistic finalization. Therefore, I believe that setting $p_{attack}=0.005$ is reasonable enough in terms of security and performance.
* Set  $\omega \geq 0.5$ (e.g., 0.5) and find $s_{ecq}$ and $s_{hcq}$ to set the epoch length $R = 2 s_{ecq} + s_{hcg}$ such that $p_{attack} \leq p$. For this we need an inition value $k_{cp}$ and find $s_{ecq}, s_{hcg}$ and $\tau$ that satisfies the three equations below:

From Theorem 6, we want that the best chain grows at least $k$ blocks. Therefore, we need 
$$(2s_{ecq} + s_{hcg})\tau = k\text{ }\text{ }\text{ }\text{ }\text{ }\text{ (1)}$$

We need $s_{ecq}$ slots to guarantee $k_{cq}$ blocks growth for the ECQ property. So, we need:

$$\tau s_{ecq} = k_{cq} \text{ }\text{ }\text{ }\text{ }\text{ }\text{ }\text{ }\text{ }\text{ (2)}$$

Lastly, we need the following as given in the Overall Result:
$$\tau = \tau_{hcg} \frac{s_{hcg}}{2 s_{ecq} + s_{hcg}}\text{ }\text{ }\text{ }\text{ }\text{ (3)}$$

Iterate $k_{cp}$ to find $s_{hcg}, s_{ecq}, \tau$ that satisfy above conditions until $p_{attack} \leq p$:
    
1.   Let $k = 4 k_{cp}$ (The CQ property parameter) We note that $4 k_{cp}$ is the optimal value that minimizes $R = 2 s_{ecq} + s_{hcg}$.
1.   $t_{hcg} = p_h  (1-c)^\D  (1-\omega)$ (to satisfy the condition in Theorem 1)
1.   $s_{hcg} = k / t_{hcg}$ (from Equation (1) and (3))
1.   $\tau = \frac{k - 2k_{cq}}{s_{hcg}}$ (from Equation (1) and (2))
1.   $s_{ecq} = k_{cq}/\tau$
1.   $p = \lceil \frac{L}{T}\rceil\frac{2^{20}(1-\alpha)n}{R}(p_{ecq} + p_{cp} + p_{cg})$

After finding $k_{cq}$ such that $p \leq p_{attack}$, let the epoch length $R = 2s_{ecq}+s_{hcg}$.

The parameters below are computed with the code in https://github.com/w3f/research/blob/master/experiments/parameters/babe_NTP.py
#### PARAMETERS OF BABE WITH NTP 
c = 0.42, slot time T = 6

It is secure in 3 years with probability 0.996248075142

It is resistant to 5.9 second network delay

-~~~~~~~~~~~~~~ Common Prefix Property ~~~~~~~~~~~~~~

k = 140

It means: Prun the last 140 blocks of the best chain. All the remaining ones are probabilistically finalized

-~~~~~~~~~~~~~~ Epoch Length ~~~~~~~~~~~~~~

Epoch length should be at least 1829 slots, 3.04833333333 hours


### BABE with the Median Algorithm
In order to make  $p_{delay}$ small, we let $\gamma = 0.1$ and $\omega = 0.37$. If we do not care about this probability, we can use the same parameters as above and specify the sync-epoch length based on the expected clock drifts.


1.   Let $k = 8 k_{cp}$ (The CQ property parameter) We note that $4 k_{cp}$ is the optimal value that minimizes $R = 2 s_{ecq} + s_{hcg}$.

Steps 2 - 6 are as above

Finding synch-epoch length

1.  Set $\mu_{hcq} = 0.54$

2. Set $2  s_{ecq} / (2  mu_{hcq} - 1)$

3. Let sync-epoch length = $s_{hcq} + 2  s_{ecq}$



The parameters below are computed with the code in https://github.com/w3f/research/blob/master/experiments/parameters/babe_median.py

#### PARAMETERS OF BABE WITH THE MEDIAN ALGORITHM 

c = 0.42, slot time T = 6

It is secure in 3 years with probability 0.995125418496

It is resistant to 2.00053609831 second network delay and0.499231950845 seconds drift in one sync-epoch

-~~~~~~~~~~~~~~ Common Prefix Property ~~~~~~~~~~~~~~

k = 312

It means: Prun the last 312 blocks of the best chain. All the remaining ones are probabilistically finalized

-~~~~~~~~~~~~~~ Epoch Length ~~~~~~~~~~~~~~

Sync-Epoch length should be at least 7188 slots, 11.98 hours

Epoch length should be at least 2130 slots,3.55 hours

Probability of waiting more than 2D in the end of sync epoch is 0.0791998132892




**Some Notes about clock drifts:** 
http://www.ntp.org/ntpfaq/NTP-s-sw-clocks-quality.htm#AEN1220
All computer clocks are not very accurate because the frequency that makes time increase is never exactly right. For example the error about 0.001% make a clock be off by almost one second per day. 
Computer clocks drift because the frequency of clocks varies over time, mostly influenced by environmental changes such as temperature, air pressure or magnetic fields, etc. Below, you can see the experiment in a non-air conditioned environment on linux computer clocks.  12 PPM correspond to one second per day roughly. I seems that in every 10000 second the change on the clocks are around 1 PPM (i.e., every 3 hours the clocks drifts 0.08 seconds.). We can roughly say that the clock drifts around 1 second per day. If we have sync epoch around 12 hours it means that we have 0.5 second drift and

[![](https://i.imgur.com/Slspcg6.png)](http://www.ntp.org/ntpfaq/NTP-s-sw-clocks-quality.htm#AEN1220)

**Figure. Frequency Correction within a Week**

## References

[1] Kiayias, Aggelos, et al. "Ouroboros: A provably secure proof-of-stake blockchain protocol." Annual International Cryptology Conference. Springer, Cham, 2017.

[2] David, Bernardo, et al. "Ouroboros praos: An adaptively-secure, semi-synchronous proof-of-stake blockchain." Annual International Conference on the Theory and Applications of Cryptographic Techniques. Springer, Cham, 2018.

[3] Badertscher, Christian, et al. "Ouroboros genesis: Composable proof-of-stake blockchains with dynamic availability." Proceedings of the 2018 ACM SIGSAC Conference on Computer and Communications Security. ACM, 2018.

[4] Aggelos Kiayias and Giorgos Panagiotakos. Speed-security tradeoffs in blockchain protocols. Cryptology ePrint Archive, Report 2015/1019, 2015. http://eprint.iacr.org/2015/1019
