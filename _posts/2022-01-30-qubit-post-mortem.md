---
layout: post
title: "Qubit Bridge Post-mortem"
author: theori
description:
categories: [ news ]
tags: [ Qubit, audit, smart contract ]
comments: true
featured: true
image: assets/images/2022-01-30/qubit_xchain.png
---

On January 28, 2022, Qubit was attacked through their cross-chain bridge. An attacker[^1] called the deposit function of the Bridge contract[^2] on Ethereum, passing in a valid resource ID[^3] that mapped to a valid BridgeHandler contract[^4]. However, this resource ID mapped to an invalid token address[^5]. As such, the token transfer function succeeded even though no tokens were actually transferred, and the deposit transaction did not revert. The bridge then notified the Bridge contract on Binance Smart Chain (BSC) of the deposit transaction and a significant number of Qubit xETH tokens were transferred to the attacker’s address on BSC.

[^1]: 0xD01Ae1A708614948B2B5e0B7AB5be6AFA01325c7
[^2]: 0x20e5e35ba29dc3b540a1aee781d0814d5c77bce6
[^3]: 0x2f422fe9ea622049d6f73f81a906b9b8cff03b7f01
[^4]: 0x17b7163cf1dbd286e262ddc68b553d899b93f526
[^5]: 0x0000000000000000000000000000000000000000

We, Theori, [audited](https://github.com/PancakeBunny-finance/qubit-finance/blob/master/audits/mound_qubit_xChain_audit_rev1.1.pdf){:target="_blank"} the cross-chain bridge code of Qubit so as soon as we heard about this attack we worked quickly with the Qubit maintainers to first understand the issue, and second to determine why this issue was not seen in our audit. The core issue is that the code provided to us during the audit did not contain the `depositETH` function and the test code we were given used the WETH contract. As such, we did not anticipate  that the Qubit admin might add an invalid token address as a valid resource to represent native ETH. Especially when considering the existing deposit function does not accept native ETH except to pay a fixed fee, our auditors assumed native ETH would not be used in the bridge.

While adding the `depositETH` function and adding an invalid token address as a valid resource seems like a small change, changing the parameters of a DeFi project should be done only with careful consideration of the assumptions of the code. Presumably under the belief that this change would not affect security, we were not consulted about these changes and we were not given the opportunity to review these changes.

There are mitigations that could have been put into place that would have prevented the attack. The `safeTransferFrom` function used by the Bridge contract could have checked that the token address is a valid contract. As the token address must be whitelisted by an admin before it will be used by this function, we did not identify this as an issue. Also, the `safeTransferFrom` function is in code that is used in other projects, such as PancakeBunny, and has been considered safe provided that the token address must be whitelisted by an admin. Similarly, the Bridge could have checked that it received tokens after calling `safeTransferFrom`.

While we cannot prevent our customers from modifying the code after our audit or changing their parameters in a way that makes their code vulnerable, additional mitigations could have been suggested by our team that would have reduced risk in this scenario. Going forward, we will strive to do a better job in suggesting defense-in-depth mitigations to our clients where appropriate.

#### References
[Protocol Exploit Report 2](https://medium.com/@QubitFin/protocol-exploit-report-2-30aade4d66de)

---
