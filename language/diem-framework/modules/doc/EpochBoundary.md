
<a name="0x1_EpochBoundary"></a>

# Module `0x1::EpochBoundary`



-  [Function `reconfigure`](#0x1_EpochBoundary_reconfigure)
-  [Function `process_fullnodes`](#0x1_EpochBoundary_process_fullnodes)
-  [Function `process_validators`](#0x1_EpochBoundary_process_validators)
-  [Function `propose_new_set`](#0x1_EpochBoundary_propose_new_set)
-  [Function `reset_counters`](#0x1_EpochBoundary_reset_counters)


<pre><code><b>use</b> <a href="Audit.md#0x1_Audit">0x1::Audit</a>;
<b>use</b> <a href="AutoPay.md#0x1_AutoPay">0x1::AutoPay</a>;
<b>use</b> <a href="Burn.md#0x1_Burn">0x1::Burn</a>;
<b>use</b> <a href="CoreAddresses.md#0x1_CoreAddresses">0x1::CoreAddresses</a>;
<b>use</b> <a href="DiemAccount.md#0x1_DiemAccount">0x1::DiemAccount</a>;
<b>use</b> <a href="DiemConfig.md#0x1_DiemConfig">0x1::DiemConfig</a>;
<b>use</b> <a href="DiemSystem.md#0x1_DiemSystem">0x1::DiemSystem</a>;
<b>use</b> <a href="Epoch.md#0x1_Epoch">0x1::Epoch</a>;
<b>use</b> <a href="../../../../../../move-stdlib/docs/FixedPoint32.md#0x1_FixedPoint32">0x1::FixedPoint32</a>;
<b>use</b> <a href="FullnodeSubsidy.md#0x1_FullnodeSubsidy">0x1::FullnodeSubsidy</a>;
<b>use</b> <a href="Globals.md#0x1_Globals">0x1::Globals</a>;
<b>use</b> <a href="NodeWeight.md#0x1_NodeWeight">0x1::NodeWeight</a>;
<b>use</b> <a href="Stats.md#0x1_Stats">0x1::Stats</a>;
<b>use</b> <a href="Subsidy.md#0x1_Subsidy">0x1::Subsidy</a>;
<b>use</b> <a href="TowerState.md#0x1_TowerState">0x1::TowerState</a>;
<b>use</b> <a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector">0x1::Vector</a>;
</code></pre>



<a name="0x1_EpochBoundary_reconfigure"></a>

## Function `reconfigure`



<pre><code><b>public</b> <b>fun</b> <a href="EpochBoundary.md#0x1_EpochBoundary_reconfigure">reconfigure</a>(vm: &signer, height_now: u64)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="EpochBoundary.md#0x1_EpochBoundary_reconfigure">reconfigure</a>(vm: &signer, height_now: u64) {
    <a href="CoreAddresses.md#0x1_CoreAddresses_assert_vm">CoreAddresses::assert_vm</a>(vm);

    <b>let</b> height_start = <a href="Epoch.md#0x1_Epoch_get_timer_height_start">Epoch::get_timer_height_start</a>(vm);

    <b>let</b> (outgoing_compliant_set, _) =
        <a href="DiemSystem.md#0x1_DiemSystem_get_fee_ratio">DiemSystem::get_fee_ratio</a>(vm, height_start, height_now);

    // NOTE: This is "nominal" because it doesn't check
    <b>let</b> compliant_nodes_count = <a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_length">Vector::length</a>(&outgoing_compliant_set);
    <b>let</b> (subsidy_units, nominal_subsidy_per) =
        <a href="Subsidy.md#0x1_Subsidy_calculate_subsidy">Subsidy::calculate_subsidy</a>(vm, compliant_nodes_count);

    <a href="EpochBoundary.md#0x1_EpochBoundary_process_fullnodes">process_fullnodes</a>(vm, nominal_subsidy_per);

    <a href="EpochBoundary.md#0x1_EpochBoundary_process_validators">process_validators</a>(vm, subsidy_units, *&outgoing_compliant_set);

    <b>let</b> proposed_set = <a href="EpochBoundary.md#0x1_EpochBoundary_propose_new_set">propose_new_set</a>(vm, height_start, height_now);

    // Update all slow wallet limits
    <b>if</b> (<a href="DiemConfig.md#0x1_DiemConfig_check_transfer_enabled">DiemConfig::check_transfer_enabled</a>()) {
        <a href="DiemAccount.md#0x1_DiemAccount_slow_wallet_epoch_drip">DiemAccount::slow_wallet_epoch_drip</a>(vm, <a href="Globals.md#0x1_Globals_get_unlock">Globals::get_unlock</a>());
        // update_validator_withdrawal_limit(vm);
    };
    <a href="EpochBoundary.md#0x1_EpochBoundary_reset_counters">reset_counters</a>(vm, proposed_set, outgoing_compliant_set, height_now)
}
</code></pre>



</details>

<a name="0x1_EpochBoundary_process_fullnodes"></a>

## Function `process_fullnodes`



<pre><code><b>fun</b> <a href="EpochBoundary.md#0x1_EpochBoundary_process_fullnodes">process_fullnodes</a>(vm: &signer, nominal_subsidy_per_node: u64)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="EpochBoundary.md#0x1_EpochBoundary_process_fullnodes">process_fullnodes</a>(vm: &signer, nominal_subsidy_per_node: u64) {
    // Fullnode subsidy
    // <b>loop</b> through validators and pay full node subsidies.
    // Should happen before transactionfees get distributed.
    // Note: need <b>to</b> check, there may be new validators which have not mined yet.
    <b>let</b> miners = <a href="TowerState.md#0x1_TowerState_get_miner_list">TowerState::get_miner_list</a>();
    // fullnode subsidy is a fraction of the total subsidy available <b>to</b> validators.
    <b>let</b> proof_price = <a href="FullnodeSubsidy.md#0x1_FullnodeSubsidy_get_proof_price">FullnodeSubsidy::get_proof_price</a>(nominal_subsidy_per_node);

    <b>let</b> k = 0;
    // Distribute mining subsidy <b>to</b> fullnodes
    <b>while</b> (k &lt; <a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_length">Vector::length</a>(&miners)) {
        <b>let</b> addr = *<a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_borrow">Vector::borrow</a>(&miners, k);
        <b>if</b> (<a href="DiemSystem.md#0x1_DiemSystem_is_validator">DiemSystem::is_validator</a>(addr)) { // skip validators
          k = k + 1;
          <b>continue</b>
        };

        // TODO: this call is repeated in propose_new_set.
        // Not sure <b>if</b> the performance hit at epoch boundary is worth the refactor.
        <b>if</b> (<a href="TowerState.md#0x1_TowerState_node_above_thresh">TowerState::node_above_thresh</a>(addr)) {
          <b>let</b> count = <a href="TowerState.md#0x1_TowerState_get_count_above_thresh_in_epoch">TowerState::get_count_above_thresh_in_epoch</a>(addr);

          <b>let</b> miner_subsidy = count * proof_price;
          <a href="FullnodeSubsidy.md#0x1_FullnodeSubsidy_distribute_fullnode_subsidy">FullnodeSubsidy::distribute_fullnode_subsidy</a>(vm, addr, miner_subsidy);
        };

        k = k + 1;
    };
}
</code></pre>



</details>

<a name="0x1_EpochBoundary_process_validators"></a>

## Function `process_validators`



<pre><code><b>fun</b> <a href="EpochBoundary.md#0x1_EpochBoundary_process_validators">process_validators</a>(vm: &signer, subsidy_units: u64, outgoing_compliant_set: vector&lt;address&gt;)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="EpochBoundary.md#0x1_EpochBoundary_process_validators">process_validators</a>(
    vm: &signer, subsidy_units: u64, outgoing_compliant_set: vector&lt;address&gt;
) {
    // Process outgoing validators:
    // Distribute Transaction fees and subsidy payments <b>to</b> all outgoing validators

    <b>if</b> (<a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_is_empty">Vector::is_empty</a>&lt;address&gt;(&outgoing_compliant_set)) <b>return</b>;

    <b>if</b> (subsidy_units &gt; 0) {
        <a href="Subsidy.md#0x1_Subsidy_process_subsidy">Subsidy::process_subsidy</a>(vm, subsidy_units, &outgoing_compliant_set);
    };

    <a href="Subsidy.md#0x1_Subsidy_process_fees">Subsidy::process_fees</a>(vm, &outgoing_compliant_set);
}
</code></pre>



</details>

<a name="0x1_EpochBoundary_propose_new_set"></a>

## Function `propose_new_set`



<pre><code><b>fun</b> <a href="EpochBoundary.md#0x1_EpochBoundary_propose_new_set">propose_new_set</a>(vm: &signer, height_start: u64, height_now: u64): vector&lt;address&gt;
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="EpochBoundary.md#0x1_EpochBoundary_propose_new_set">propose_new_set</a>(vm: &signer, height_start: u64, height_now: u64): vector&lt;address&gt; {
    // Propose upcoming validator set:
    // Step 1: Sort Top N eligible validators
    // Step 2: Jail non-performing validators
    // Step 3: Reset counters
    // Step 4: Bulk <b>update</b> validator set (reconfig)

    // save all the eligible list, before the jailing removes them.
    <b>let</b> proposed_set = <a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_empty">Vector::empty</a>();

    <b>let</b> top_accounts = <a href="NodeWeight.md#0x1_NodeWeight_top_n_accounts">NodeWeight::top_n_accounts</a>(
        vm, <a href="Globals.md#0x1_Globals_get_max_validators_per_set">Globals::get_max_validators_per_set</a>()
    );

    <b>let</b> jailed_set = <a href="DiemSystem.md#0x1_DiemSystem_get_jailed_set">DiemSystem::get_jailed_set</a>(vm, height_start, height_now);

    <a href="Burn.md#0x1_Burn_reset_ratios">Burn::reset_ratios</a>(vm);
    // LEAVE THIS CODE COMMENTED for future <b>use</b>
    // TODO: Make the burn value dynamic.
    // <b>let</b> incoming_count = <a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_length">Vector::length</a>&lt;address&gt;(&top_accounts) - <a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_length">Vector::length</a>&lt;address&gt;(&jailed_set);
    // <b>let</b> burn_value = <a href="Subsidy.md#0x1_Subsidy_subsidy_curve">Subsidy::subsidy_curve</a>(
    //   <a href="Globals.md#0x1_Globals_get_subsidy_ceiling_gas">Globals::get_subsidy_ceiling_gas</a>(),
    //   incoming_count,
    //   Globals::get_max_node_density()
    // )/4;

    <b>let</b> burn_value = 1000000; // TODO: switch <b>to</b> a variable cost, <b>as</b> above.

    <b>let</b> i = 0;
    <b>while</b> (i &lt; <a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_length">Vector::length</a>&lt;address&gt;(&top_accounts)) {
        <b>let</b> addr = *<a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_borrow">Vector::borrow</a>(&top_accounts, i);
        <b>let</b> mined_last_epoch = <a href="TowerState.md#0x1_TowerState_node_above_thresh">TowerState::node_above_thresh</a>(addr);
        // TODO: temporary until jailing is enabled.
        <b>if</b> (
            !<a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_contains">Vector::contains</a>(&jailed_set, &addr) &&
            mined_last_epoch &&
            <a href="Audit.md#0x1_Audit_val_audit_passing">Audit::val_audit_passing</a>(addr)
        ) {
            <a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_push_back">Vector::push_back</a>(&<b>mut</b> proposed_set, addr);
            <a href="Burn.md#0x1_Burn_epoch_start_burn">Burn::epoch_start_burn</a>(vm, addr, burn_value);
        };
        i = i+ 1;
    };

    // If the cardinality of validator_set in the next epoch is less than 4,
    // we keep the same validator set.
    <b>if</b> (<a href="../../../../../../move-stdlib/docs/Vector.md#0x1_Vector_length">Vector::length</a>&lt;address&gt;(&proposed_set) &lt;= 3) proposed_set = *&top_accounts;
    // Usually an issue in staging network for QA only.
    // This is very rare and theoretically impossible for network <b>with</b>
    // at least 6 nodes and 6 rounds. If we reach an epoch boundary <b>with</b>
    // at least 6 rounds, we would have at least 2/3rd of the validator
    // set <b>with</b> at least 66% liveliness.
    proposed_set
}
</code></pre>



</details>

<a name="0x1_EpochBoundary_reset_counters"></a>

## Function `reset_counters`



<pre><code><b>fun</b> <a href="EpochBoundary.md#0x1_EpochBoundary_reset_counters">reset_counters</a>(vm: &signer, proposed_set: vector&lt;address&gt;, outgoing_compliant: vector&lt;address&gt;, height_now: u64)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="EpochBoundary.md#0x1_EpochBoundary_reset_counters">reset_counters</a>(vm: &signer, proposed_set: vector&lt;address&gt;, outgoing_compliant: vector&lt;address&gt;, height_now: u64) {

    // Reset <a href="Stats.md#0x1_Stats">Stats</a>
    <a href="Stats.md#0x1_Stats_reconfig">Stats::reconfig</a>(vm, &proposed_set);

    // Migrate <a href="TowerState.md#0x1_TowerState">TowerState</a> list from elegible.
    <a href="TowerState.md#0x1_TowerState_reconfig">TowerState::reconfig</a>(vm, &outgoing_compliant);

    // Reconfigure the network
    <a href="DiemSystem.md#0x1_DiemSystem_bulk_update_validators">DiemSystem::bulk_update_validators</a>(vm, proposed_set);

    // process community wallets
    <a href="DiemAccount.md#0x1_DiemAccount_process_community_wallets">DiemAccount::process_community_wallets</a>(vm, <a href="DiemConfig.md#0x1_DiemConfig_get_current_epoch">DiemConfig::get_current_epoch</a>());

    // reset counters
    <a href="AutoPay.md#0x1_AutoPay_reconfig_reset_tick">AutoPay::reconfig_reset_tick</a>(vm);
    <a href="Epoch.md#0x1_Epoch_reset_timer">Epoch::reset_timer</a>(vm, height_now);
}
</code></pre>



</details>


[//]: # ("File containing references which can be used from documentation")
[ACCESS_CONTROL]: https://github.com/diem/dip/blob/main/dips/dip-2.md
[ROLE]: https://github.com/diem/dip/blob/main/dips/dip-2.md#roles
[PERMISSION]: https://github.com/diem/dip/blob/main/dips/dip-2.md#permissions
