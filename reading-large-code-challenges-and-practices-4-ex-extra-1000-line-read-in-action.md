---
date: "2024-12-02T16:49:46.8051208+08:00"
lang: zh
slug: reading-large-code-challenges-and-practices-4-ex-extra-1000-line-read-in-action
title: 阅读大规模代码：挑战与实践（4-EX）番外篇：试炼阅读某1000行的函数
---
上一章我们讲了怎么利用反向追踪的方法读代码。但是示范的代码太短了，实战中，一个函数几百上千行很常见（即使在一些比较优秀的开源项目中也可能存在）。

这章我们将带你阅读一个1000行的函数，不过这么长的代码，抱有目的地去读会比较有效率。

下面来阅读这个函数，摘自 rust-lightning 的 `ChannelManager::read` 函数。它的作用是从磁盘等任何二进制数据源中恢复一个 `ChannelManager` 对象。

我们设定的目标是，**理解它到底从磁盘读了哪些东西出来**，以便分析这些数据存放在磁盘是否安全。

> 设立一个合理的目标很重要。一千行代码都够有的语言实现一套编译器了，没有必要浪费时间在普通的数据系统项目上。

## 代码

请自己试一试。

```rust
impl<'a, M: Deref, T: Deref, ES: Deref, NS: Deref, SP: Deref, F: Deref, R: Deref, L: Deref>
	ReadableArgs<ChannelManagerReadArgs<'a, M, T, ES, NS, SP, F, R, L>>
	for (BlockHash, ChannelManager<M, T, ES, NS, SP, F, R, L>)
where
	M::Target: chain::Watch<<SP::Target as SignerProvider>::EcdsaSigner>,
	T::Target: BroadcasterInterface,
	ES::Target: EntropySource,
	NS::Target: NodeSigner,
	SP::Target: SignerProvider,
	F::Target: FeeEstimator,
	R::Target: Router,
	L::Target: Logger,
{
	fn read<Reader: io::Read>(
		reader: &mut Reader, mut args: ChannelManagerReadArgs<'a, M, T, ES, NS, SP, F, R, L>,
	) -> Result<Self, DecodeError> {
		let _ver = read_ver_prefix!(reader, SERIALIZATION_VERSION);

		let chain_hash: ChainHash = Readable::read(reader)?;
		let best_block_height: u32 = Readable::read(reader)?;
		let best_block_hash: BlockHash = Readable::read(reader)?;

		let mut failed_htlcs = Vec::new();

		let channel_count: u64 = Readable::read(reader)?;
		let mut funding_txo_set = hash_set_with_capacity(cmp::min(channel_count as usize, 128));
		let mut funded_peer_channels: HashMap<PublicKey, HashMap<ChannelId, ChannelPhase<SP>>> =
			hash_map_with_capacity(cmp::min(channel_count as usize, 128));
		let mut outpoint_to_peer = hash_map_with_capacity(cmp::min(channel_count as usize, 128));
		let mut short_to_chan_info = hash_map_with_capacity(cmp::min(channel_count as usize, 128));
		let mut channel_closures = VecDeque::new();
		let mut close_background_events = Vec::new();
		let mut funding_txo_to_channel_id = hash_map_with_capacity(channel_count as usize);
		for _ in 0..channel_count {
			let mut channel: Channel<SP> = Channel::read(
				reader,
				(
					&args.entropy_source,
					&args.signer_provider,
					best_block_height,
					&provided_channel_type_features(&args.default_config),
					args.color_source.clone(),
				),
			)?;
			let logger = WithChannelContext::from(&args.logger, &channel.context);
			let funding_txo = channel.context.get_funding_txo().ok_or(DecodeError::InvalidValue)?;
			funding_txo_to_channel_id.insert(funding_txo, channel.context.channel_id());
			funding_txo_set.insert(funding_txo.clone());
			if let Some(ref mut monitor) = args.channel_monitors.get_mut(&funding_txo) {
				if channel.get_cur_holder_commitment_transaction_number()
					> monitor.get_cur_holder_commitment_number()
					|| channel.get_revoked_counterparty_commitment_transaction_number()
						> monitor.get_min_seen_secret()
					|| channel.get_cur_counterparty_commitment_transaction_number()
						> monitor.get_cur_counterparty_commitment_number()
					|| channel.context.get_latest_monitor_update_id()
						< monitor.get_latest_update_id()
				{
					// But if the channel is behind of the monitor, close the channel:
					log_error!(
						logger,
						"A ChannelManager is stale compared to the current ChannelMonitor!"
					);
					log_error!(logger, " The channel will be force-closed and the latest commitment transaction from the ChannelMonitor broadcast.");
					if channel.context.get_latest_monitor_update_id()
						< monitor.get_latest_update_id()
					{
						log_error!(logger, " The ChannelMonitor for channel {} is at update_id {} but the ChannelManager is at update_id {}.",
							&channel.context.channel_id(), monitor.get_latest_update_id(), channel.context.get_latest_monitor_update_id());
					}
					if channel.get_cur_holder_commitment_transaction_number()
						> monitor.get_cur_holder_commitment_number()
					{
						log_error!(logger, " The ChannelMonitor for channel {} is at holder commitment number {} but the ChannelManager is at holder commitment number {}.",
							&channel.context.channel_id(), monitor.get_cur_holder_commitment_number(), channel.get_cur_holder_commitment_transaction_number());
					}
					if channel.get_revoked_counterparty_commitment_transaction_number()
						> monitor.get_min_seen_secret()
					{
						log_error!(logger, " The ChannelMonitor for channel {} is at revoked counterparty transaction number {} but the ChannelManager is at revoked counterparty transaction number {}.",
							&channel.context.channel_id(), monitor.get_min_seen_secret(), channel.get_revoked_counterparty_commitment_transaction_number());
					}
					if channel.get_cur_counterparty_commitment_transaction_number()
						> monitor.get_cur_counterparty_commitment_number()
					{
						log_error!(logger, " The ChannelMonitor for channel {} is at counterparty commitment transaction number {} but the ChannelManager is at counterparty commitment transaction number {}.",
							&channel.context.channel_id(), monitor.get_cur_counterparty_commitment_number(), channel.get_cur_counterparty_commitment_transaction_number());
					}
					let mut shutdown_result =
						channel.context.force_shutdown(true, ClosureReason::OutdatedChannelManager);
					if shutdown_result.unbroadcasted_batch_funding_txid.is_some() {
						return Err(DecodeError::InvalidValue);
					}
					if let Some((counterparty_node_id, funding_txo, channel_id, update)) =
						shutdown_result.monitor_update
					{
						close_background_events.push(
							BackgroundEvent::MonitorUpdateRegeneratedOnStartup {
								counterparty_node_id,
								funding_txo,
								channel_id,
								update,
							},
						);
					}
					failed_htlcs.append(&mut shutdown_result.dropped_outbound_htlcs);
					channel_closures.push_back((
						events::Event::ChannelClosed {
							channel_id: channel.context.channel_id(),
							user_channel_id: channel.context.get_user_id(),
							reason: ClosureReason::OutdatedChannelManager,
							counterparty_node_id: Some(channel.context.get_counterparty_node_id()),
							channel_capacity_sats: Some(channel.context.get_value_satoshis()),
							channel_funding_txo: channel.context.get_funding_txo(),
						},
						None,
					));
					for (channel_htlc_source, payment_hash) in channel.inflight_htlc_sources() {
						let mut found_htlc = false;
						for (monitor_htlc_source, _) in monitor.get_all_current_outbound_htlcs() {
							if *channel_htlc_source == monitor_htlc_source {
								found_htlc = true;
								break;
							}
						}
						if !found_htlc {
							// If we have some HTLCs in the channel which are not present in the newer
							// ChannelMonitor, they have been removed and should be failed back to
							// ensure we don't forget them entirely. Note that if the missing HTLC(s)
							// were actually claimed we'd have generated and ensured the previous-hop
							// claim update ChannelMonitor updates were persisted prior to persising
							// the ChannelMonitor update for the forward leg, so attempting to fail the
							// backwards leg of the HTLC will simply be rejected.
							log_info!(logger,
								"Failing HTLC with hash {} as it is missing in the ChannelMonitor for channel {} but was present in the (stale) ChannelManager",
								&channel.context.channel_id(), &payment_hash);
							failed_htlcs.push((
								channel_htlc_source.clone(),
								*payment_hash,
								channel.context.get_counterparty_node_id(),
								channel.context.channel_id(),
							));
						}
					}
				} else {
					channel.on_startup_drop_completed_blocked_mon_updates_through(
						&logger,
						monitor.get_latest_update_id(),
					);
					log_info!(logger, "Successfully loaded channel {} at update_id {} against monitor at update id {} with {} blocked updates",
						&channel.context.channel_id(), channel.context.get_latest_monitor_update_id(),
						monitor.get_latest_update_id(), channel.blocked_monitor_updates_pending());
					if let Some(short_channel_id) = channel.context.get_short_channel_id() {
						short_to_chan_info.insert(
							short_channel_id,
							(
								channel.context.get_counterparty_node_id(),
								channel.context.channel_id(),
							),
						);
					}
					if let Some(funding_txo) = channel.context.get_funding_txo() {
						outpoint_to_peer
							.insert(funding_txo, channel.context.get_counterparty_node_id());
					}
					match funded_peer_channels.entry(channel.context.get_counterparty_node_id()) {
						hash_map::Entry::Occupied(mut entry) => {
							let by_id_map = entry.get_mut();
							by_id_map.insert(
								channel.context.channel_id(),
								ChannelPhase::Funded(channel),
							);
						},
						hash_map::Entry::Vacant(entry) => {
							let mut by_id_map = new_hash_map();
							by_id_map.insert(
								channel.context.channel_id(),
								ChannelPhase::Funded(channel),
							);
							entry.insert(by_id_map);
						},
					}
				}
			} else if channel.is_awaiting_initial_mon_persist() {
				// If we were persisted and shut down while the initial ChannelMonitor persistence
				// was in-progress, we never broadcasted the funding transaction and can still
				// safely discard the channel.
				let _ = channel.context.force_shutdown(false, ClosureReason::DisconnectedPeer);
				channel_closures.push_back((
					events::Event::ChannelClosed {
						channel_id: channel.context.channel_id(),
						user_channel_id: channel.context.get_user_id(),
						reason: ClosureReason::DisconnectedPeer,
						counterparty_node_id: Some(channel.context.get_counterparty_node_id()),
						channel_capacity_sats: Some(channel.context.get_value_satoshis()),
						channel_funding_txo: channel.context.get_funding_txo(),
					},
					None,
				));
			} else {
				log_error!(
					logger,
					"Missing ChannelMonitor for channel {} needed by ChannelManager.",
					&channel.context.channel_id()
				);
				log_error!(logger, " The chain::Watch API *requires* that monitors are persisted durably before returning,");
				log_error!(logger, " client applications must ensure that ChannelMonitor data is always available and the latest to avoid funds loss!");
				log_error!(
					logger,
					" Without the ChannelMonitor we cannot continue without risking funds."
				);
				log_error!(logger, " Please ensure the chain::Watch API requirements are met and file a bug report at https://github.com/lightningdevkit/rust-lightning");
				return Err(DecodeError::InvalidValue);
			}
		}

		for (funding_txo, monitor) in args.channel_monitors.iter() {
			if !funding_txo_set.contains(funding_txo) {
				let logger = WithChannelMonitor::from(&args.logger, monitor);
				let channel_id = monitor.channel_id();
				log_info!(
					logger,
					"Queueing monitor update to ensure missing channel {} is force closed",
					&channel_id
				);
				let monitor_update = ChannelMonitorUpdate {
					update_id: CLOSED_CHANNEL_UPDATE_ID,
					counterparty_node_id: None,
					updates: vec![ChannelMonitorUpdateStep::ChannelForceClosed {
						should_broadcast: true,
					}],
					channel_id: Some(monitor.channel_id()),
				};
				close_background_events.push(
					BackgroundEvent::ClosedMonitorUpdateRegeneratedOnStartup((
						*funding_txo,
						channel_id,
						monitor_update,
					)),
				);
			}
		}

		const MAX_ALLOC_SIZE: usize = 1024 * 64;
		let forward_htlcs_count: u64 = Readable::read(reader)?;
		let mut forward_htlcs = hash_map_with_capacity(cmp::min(forward_htlcs_count as usize, 128));
		for _ in 0..forward_htlcs_count {
			let short_channel_id = Readable::read(reader)?;
			let pending_forwards_count: u64 = Readable::read(reader)?;
			let mut pending_forwards = Vec::with_capacity(cmp::min(
				pending_forwards_count as usize,
				MAX_ALLOC_SIZE / mem::size_of::<HTLCForwardInfo>(),
			));
			for _ in 0..pending_forwards_count {
				pending_forwards.push(Readable::read(reader)?);
			}
			forward_htlcs.insert(short_channel_id, pending_forwards);
		}

		let claimable_htlcs_count: u64 = Readable::read(reader)?;
		let mut claimable_htlcs_list =
			Vec::with_capacity(cmp::min(claimable_htlcs_count as usize, 128));
		for _ in 0..claimable_htlcs_count {
			let payment_hash = Readable::read(reader)?;
			let previous_hops_len: u64 = Readable::read(reader)?;
			let mut previous_hops = Vec::with_capacity(cmp::min(
				previous_hops_len as usize,
				MAX_ALLOC_SIZE / mem::size_of::<ClaimableHTLC>(),
			));
			for _ in 0..previous_hops_len {
				previous_hops.push(<ClaimableHTLC as Readable>::read(reader)?);
			}
			claimable_htlcs_list.push((payment_hash, previous_hops));
		}

		let peer_state_from_chans = |channel_by_id| PeerState {
			channel_by_id,
			inbound_channel_request_by_id: new_hash_map(),
			latest_features: InitFeatures::empty(),
			pending_msg_events: Vec::new(),
			in_flight_monitor_updates: BTreeMap::new(),
			monitor_update_blocked_actions: BTreeMap::new(),
			actions_blocking_raa_monitor_updates: BTreeMap::new(),
			is_connected: false,
		};

		let peer_count: u64 = Readable::read(reader)?;
		let mut per_peer_state = hash_map_with_capacity(cmp::min(
			peer_count as usize,
			MAX_ALLOC_SIZE / mem::size_of::<(PublicKey, Mutex<PeerState<SP>>)>(),
		));
		for _ in 0..peer_count {
			let peer_pubkey = Readable::read(reader)?;
			let peer_chans = funded_peer_channels.remove(&peer_pubkey).unwrap_or(new_hash_map());
			let mut peer_state = peer_state_from_chans(peer_chans);
			peer_state.latest_features = Readable::read(reader)?;
			per_peer_state.insert(peer_pubkey, Mutex::new(peer_state));
		}

		let event_count: u64 = Readable::read(reader)?;
		let mut pending_events_read: VecDeque<(events::Event, Option<EventCompletionAction>)> =
			VecDeque::with_capacity(cmp::min(
				event_count as usize,
				MAX_ALLOC_SIZE / mem::size_of::<(events::Event, Option<EventCompletionAction>)>(),
			));
		for _ in 0..event_count {
			match MaybeReadable::read(reader)? {
				Some(event) => pending_events_read.push_back((event, None)),
				None => continue,
			}
		}

		let background_event_count: u64 = Readable::read(reader)?;
		for _ in 0..background_event_count {
			match <u8 as Readable>::read(reader)? {
				0 => {
					// LDK versions prior to 0.0.116 wrote pending `MonitorUpdateRegeneratedOnStartup`s here,
					// however we really don't (and never did) need them - we regenerate all
					// on-startup monitor updates.
					let _: OutPoint = Readable::read(reader)?;
					let _: ChannelMonitorUpdate = Readable::read(reader)?;
				},
				_ => return Err(DecodeError::InvalidValue),
			}
		}

		let _last_node_announcement_serial: u32 = Readable::read(reader)?; // Only used < 0.0.111
		let highest_seen_timestamp: u32 = Readable::read(reader)?;

		let pending_inbound_payment_count: u64 = Readable::read(reader)?;
		let mut pending_inbound_payments: HashMap<PaymentHash, PendingInboundPayment> =
			hash_map_with_capacity(cmp::min(
				pending_inbound_payment_count as usize,
				MAX_ALLOC_SIZE / (3 * 32),
			));
		for _ in 0..pending_inbound_payment_count {
			if pending_inbound_payments
				.insert(Readable::read(reader)?, Readable::read(reader)?)
				.is_some()
			{
				return Err(DecodeError::InvalidValue);
			}
		}

		let pending_outbound_payments_count_compat: u64 = Readable::read(reader)?;
		let mut pending_outbound_payments_compat: HashMap<PaymentId, PendingOutboundPayment> =
			hash_map_with_capacity(cmp::min(
				pending_outbound_payments_count_compat as usize,
				MAX_ALLOC_SIZE / 32,
			));
		for _ in 0..pending_outbound_payments_count_compat {
			let session_priv = Readable::read(reader)?;
			let payment = PendingOutboundPayment::Legacy {
				session_privs: hash_set_from_iter([session_priv]),
			};
			if pending_outbound_payments_compat.insert(PaymentId(session_priv), payment).is_some() {
				return Err(DecodeError::InvalidValue);
			};
		}

		// pending_outbound_payments_no_retry is for compatibility with 0.0.101 clients.
		let mut pending_outbound_payments_no_retry: Option<HashMap<PaymentId, HashSet<[u8; 32]>>> =
			None;
		let mut pending_outbound_payments = None;
		let mut pending_intercepted_htlcs: Option<HashMap<InterceptId, PendingAddHTLCInfo>> =
			Some(new_hash_map());
		let mut received_network_pubkey: Option<PublicKey> = None;
		let mut fake_scid_rand_bytes: Option<[u8; 32]> = None;
		let mut probing_cookie_secret: Option<[u8; 32]> = None;
		let mut claimable_htlc_purposes = None;
		let mut claimable_htlc_onion_fields = None;
		let mut pending_claiming_payments = Some(new_hash_map());
		let mut monitor_update_blocked_actions_per_peer: Option<Vec<(_, BTreeMap<_, Vec<_>>)>> =
			Some(Vec::new());
		let mut events_override = None;
		let mut in_flight_monitor_updates: Option<
			HashMap<(PublicKey, OutPoint), Vec<ChannelMonitorUpdate>>,
		> = None;
		let mut decode_update_add_htlcs: Option<HashMap<u64, Vec<msgs::UpdateAddHTLC>>> = None;
		read_tlv_fields!(reader, {
			(1, pending_outbound_payments_no_retry, option),
			(2, pending_intercepted_htlcs, option),
			(3, pending_outbound_payments, option),
			(4, pending_claiming_payments, option),
			(5, received_network_pubkey, option),
			(6, monitor_update_blocked_actions_per_peer, option),
			(7, fake_scid_rand_bytes, option),
			(8, events_override, option),
			(9, claimable_htlc_purposes, optional_vec),
			(10, in_flight_monitor_updates, option),
			(11, probing_cookie_secret, option),
			(13, claimable_htlc_onion_fields, optional_vec),
			(14, decode_update_add_htlcs, option),
		});
		let mut decode_update_add_htlcs = decode_update_add_htlcs.unwrap_or_else(|| new_hash_map());
		if fake_scid_rand_bytes.is_none() {
			fake_scid_rand_bytes = Some(args.entropy_source.get_secure_random_bytes());
		}

		if probing_cookie_secret.is_none() {
			probing_cookie_secret = Some(args.entropy_source.get_secure_random_bytes());
		}

		if let Some(events) = events_override {
			pending_events_read = events;
		}

		if !channel_closures.is_empty() {
			pending_events_read.append(&mut channel_closures);
		}

		if pending_outbound_payments.is_none() && pending_outbound_payments_no_retry.is_none() {
			pending_outbound_payments = Some(pending_outbound_payments_compat);
		} else if pending_outbound_payments.is_none() {
			let mut outbounds = new_hash_map();
			for (id, session_privs) in pending_outbound_payments_no_retry.unwrap().drain() {
				outbounds.insert(id, PendingOutboundPayment::Legacy { session_privs });
			}
			pending_outbound_payments = Some(outbounds);
		}
		let pending_outbounds = OutboundPayments {
			pending_outbound_payments: Mutex::new(pending_outbound_payments.unwrap()),
			retry_lock: Mutex::new(()),
			color_source: args.color_source.clone(),
		};

		// We have to replay (or skip, if they were completed after we wrote the `ChannelManager`)
		// each `ChannelMonitorUpdate` in `in_flight_monitor_updates`. After doing so, we have to
		// check that each channel we have isn't newer than the latest `ChannelMonitorUpdate`(s) we
		// replayed, and for each monitor update we have to replay we have to ensure there's a
		// `ChannelMonitor` for it.
		//
		// In order to do so we first walk all of our live channels (so that we can check their
		// state immediately after doing the update replays, when we have the `update_id`s
		// available) and then walk any remaining in-flight updates.
		//
		// Because the actual handling of the in-flight updates is the same, it's macro'ized here:
		let mut pending_background_events = Vec::new();
		macro_rules! handle_in_flight_updates {
			($counterparty_node_id: expr, $chan_in_flight_upds: expr, $funding_txo: expr,
			 $monitor: expr, $peer_state: expr, $logger: expr, $channel_info_log: expr
			) => {{
				let mut max_in_flight_update_id = 0;
				$chan_in_flight_upds.retain(|upd| upd.update_id > $monitor.get_latest_update_id());
				for update in $chan_in_flight_upds.iter() {
					log_trace!(
						$logger,
						"Replaying ChannelMonitorUpdate {} for {}channel {}",
						update.update_id,
						$channel_info_log,
						&$monitor.channel_id()
					);
					max_in_flight_update_id = cmp::max(max_in_flight_update_id, update.update_id);
					pending_background_events.push(
						BackgroundEvent::MonitorUpdateRegeneratedOnStartup {
							counterparty_node_id: $counterparty_node_id,
							funding_txo: $funding_txo,
							channel_id: $monitor.channel_id(),
							update: update.clone(),
						},
					);
				}
				if $chan_in_flight_upds.is_empty() {
					// We had some updates to apply, but it turns out they had completed before we
					// were serialized, we just weren't notified of that. Thus, we may have to run
					// the completion actions for any monitor updates, but otherwise are done.
					pending_background_events.push(BackgroundEvent::MonitorUpdatesComplete {
						counterparty_node_id: $counterparty_node_id,
						channel_id: $monitor.channel_id(),
					});
				}
				if $peer_state
					.in_flight_monitor_updates
					.insert($funding_txo, $chan_in_flight_upds)
					.is_some()
				{
					log_error!(
						$logger,
						"Duplicate in-flight monitor update set for the same channel!"
					);
					return Err(DecodeError::InvalidValue);
				}
				max_in_flight_update_id
			}};
		}

		for (counterparty_id, peer_state_mtx) in per_peer_state.iter_mut() {
			let mut peer_state_lock = peer_state_mtx.lock().unwrap();
			let peer_state = &mut *peer_state_lock;
			for phase in peer_state.channel_by_id.values() {
				if let ChannelPhase::Funded(chan) = phase {
					let logger = WithChannelContext::from(&args.logger, &chan.context);

					// Channels that were persisted have to be funded, otherwise they should have been
					// discarded.
					let funding_txo =
						chan.context.get_funding_txo().ok_or(DecodeError::InvalidValue)?;
					let monitor = args
						.channel_monitors
						.get(&funding_txo)
						.expect("We already checked for monitor presence when loading channels");
					let mut max_in_flight_update_id = monitor.get_latest_update_id();
					if let Some(in_flight_upds) = &mut in_flight_monitor_updates {
						if let Some(mut chan_in_flight_upds) =
							in_flight_upds.remove(&(*counterparty_id, funding_txo))
						{
							max_in_flight_update_id = cmp::max(
								max_in_flight_update_id,
								handle_in_flight_updates!(
									*counterparty_id,
									chan_in_flight_upds,
									funding_txo,
									monitor,
									peer_state,
									logger,
									""
								),
							);
						}
					}
					if chan.get_latest_unblocked_monitor_update_id() > max_in_flight_update_id {
						// If the channel is ahead of the monitor, return DangerousValue:
						log_error!(logger, "A ChannelMonitor is stale compared to the current ChannelManager! This indicates a potentially-critical violation of the chain::Watch API!");
						log_error!(logger, " The ChannelMonitor for channel {} is at update_id {} with update_id through {} in-flight",
							chan.context.channel_id(), monitor.get_latest_update_id(), max_in_flight_update_id);
						log_error!(
							logger,
							" but the ChannelManager is at update_id {}.",
							chan.get_latest_unblocked_monitor_update_id()
						);
						log_error!(logger, " The chain::Watch API *requires* that monitors are persisted durably before returning,");
						log_error!(logger, " client applications must ensure that ChannelMonitor data is always available and the latest to avoid funds loss!");
						log_error!(logger, " Without the latest ChannelMonitor we cannot continue without risking funds.");
						log_error!(logger, " Please ensure the chain::Watch API requirements are met and file a bug report at https://github.com/lightningdevkit/rust-lightning");
						return Err(DecodeError::DangerousValue);
					}
				} else {
					// We shouldn't have persisted (or read) any unfunded channel types so none should have been
					// created in this `channel_by_id` map.
					debug_assert!(false);
					return Err(DecodeError::InvalidValue);
				}
			}
		}

		if let Some(in_flight_upds) = in_flight_monitor_updates {
			for ((counterparty_id, funding_txo), mut chan_in_flight_updates) in in_flight_upds {
				let channel_id = funding_txo_to_channel_id.get(&funding_txo).copied();
				let logger = WithContext::from(&args.logger, Some(counterparty_id), channel_id);
				if let Some(monitor) = args.channel_monitors.get(&funding_txo) {
					// Now that we've removed all the in-flight monitor updates for channels that are
					// still open, we need to replay any monitor updates that are for closed channels,
					// creating the neccessary peer_state entries as we go.
					let peer_state_mutex = per_peer_state
						.entry(counterparty_id)
						.or_insert_with(|| Mutex::new(peer_state_from_chans(new_hash_map())));
					let mut peer_state = peer_state_mutex.lock().unwrap();
					handle_in_flight_updates!(
						counterparty_id,
						chan_in_flight_updates,
						funding_txo,
						monitor,
						peer_state,
						logger,
						"closed "
					);
				} else {
					log_error!(logger, "A ChannelMonitor is missing even though we have in-flight updates for it! This indicates a potentially-critical violation of the chain::Watch API!");
					log_error!(
						logger,
						" The ChannelMonitor for channel {} is missing.",
						if let Some(channel_id) = channel_id {
							channel_id.to_string()
						} else {
							format!("with outpoint {}", funding_txo)
						}
					);
					log_error!(logger, " The chain::Watch API *requires* that monitors are persisted durably before returning,");
					log_error!(logger, " client applications must ensure that ChannelMonitor data is always available and the latest to avoid funds loss!");
					log_error!(logger, " Without the latest ChannelMonitor we cannot continue without risking funds.");
					log_error!(logger, " Please ensure the chain::Watch API requirements are met and file a bug report at https://github.com/lightningdevkit/rust-lightning");
					log_error!(
						logger,
						" Pending in-flight updates are: {:?}",
						chan_in_flight_updates
					);
					return Err(DecodeError::InvalidValue);
				}
			}
		}

		// Note that we have to do the above replays before we push new monitor updates.
		pending_background_events.append(&mut close_background_events);

		// If there's any preimages for forwarded HTLCs hanging around in ChannelMonitors we
		// should ensure we try them again on the inbound edge. We put them here and do so after we
		// have a fully-constructed `ChannelManager` at the end.
		let mut pending_claims_to_replay = Vec::new();

		{
			// If we're tracking pending payments, ensure we haven't lost any by looking at the
			// ChannelMonitor data for any channels for which we do not have authorative state
			// (i.e. those for which we just force-closed above or we otherwise don't have a
			// corresponding `Channel` at all).
			// This avoids several edge-cases where we would otherwise "forget" about pending
			// payments which are still in-flight via their on-chain state.
			// We only rebuild the pending payments map if we were most recently serialized by
			// 0.0.102+
			for (_, monitor) in args.channel_monitors.iter() {
				let counterparty_opt = outpoint_to_peer.get(&monitor.get_funding_txo().0);
				if counterparty_opt.is_none() {
					let logger = WithChannelMonitor::from(&args.logger, monitor);
					for (htlc_source, (htlc, _)) in monitor.get_pending_or_resolved_outbound_htlcs()
					{
						if let HTLCSource::OutboundRoute {
							payment_id, session_priv, path, ..
						} = htlc_source
						{
							if path.hops.is_empty() {
								log_error!(logger, "Got an empty path for a pending payment");
								return Err(DecodeError::InvalidValue);
							}

							let path_amt = path.final_value_msat();
							let mut session_priv_bytes = [0; 32];
							session_priv_bytes[..].copy_from_slice(&session_priv[..]);
							match pending_outbounds
								.pending_outbound_payments
								.lock()
								.unwrap()
								.entry(payment_id)
							{
								hash_map::Entry::Occupied(mut entry) => {
									let newly_added =
										entry.get_mut().insert(session_priv_bytes, &path);
									log_info!(logger, "{} a pending payment path for {} msat for session priv {} on an existing pending payment with payment hash {}",
										if newly_added { "Added" } else { "Had" }, path_amt, log_bytes!(session_priv_bytes), htlc.payment_hash);
								},
								hash_map::Entry::Vacant(entry) => {
									let path_fee = path.fee_msat();
									entry.insert(PendingOutboundPayment::Retryable {
										retry_strategy: None,
										attempts: PaymentAttempts::new(),
										payment_params: None,
										session_privs: hash_set_from_iter([session_priv_bytes]),
										payment_hash: htlc.payment_hash,
										payment_secret: None, // only used for retries, and we'll never retry on startup
										payment_metadata: None, // only used for retries, and we'll never retry on startup
										keysend_preimage: None, // only used for retries, and we'll never retry on startup
										custom_tlvs: Vec::new(), // only used for retries, and we'll never retry on startup
										pending_amt_msat: path_amt,
										pending_fee_msat: Some(path_fee),
										total_msat: path_amt,
										starting_block_height: best_block_height,
										remaining_max_total_routing_fee_msat: None, // only used for retries, and we'll never retry on startup
									});
									log_info!(logger, "Added a pending payment for {} msat with payment hash {} for path with session priv {}",
										path_amt, &htlc.payment_hash,  log_bytes!(session_priv_bytes));
								},
							}
						}
					}
					for (htlc_source, (htlc, preimage_opt)) in
						monitor.get_all_current_outbound_htlcs()
					{
						match htlc_source {
							HTLCSource::PreviousHopData(prev_hop_data) => {
								let pending_forward_matches_htlc = |info: &PendingAddHTLCInfo| {
									info.prev_funding_outpoint == prev_hop_data.outpoint
										&& info.prev_htlc_id == prev_hop_data.htlc_id
								};
								// The ChannelMonitor is now responsible for this HTLC's
								// failure/success and will let us know what its outcome is. If we
								// still have an entry for this HTLC in `forward_htlcs` or
								// `pending_intercepted_htlcs`, we were apparently not persisted after
								// the monitor was when forwarding the payment.
								decode_update_add_htlcs.retain(|scid, update_add_htlcs| {
									update_add_htlcs.retain(|update_add_htlc| {
										let matches = *scid == prev_hop_data.short_channel_id &&
											update_add_htlc.htlc_id == prev_hop_data.htlc_id;
										if matches {
											log_info!(logger, "Removing pending to-decode HTLC with hash {} as it was forwarded to the closed channel {}",
												&htlc.payment_hash, &monitor.channel_id());
										}
										!matches
									});
									!update_add_htlcs.is_empty()
								});
								forward_htlcs.retain(|_, forwards| {
									forwards.retain(|forward| {
										if let HTLCForwardInfo::AddHTLC(htlc_info) = forward {
											if pending_forward_matches_htlc(&htlc_info) {
												log_info!(logger, "Removing pending to-forward HTLC with hash {} as it was forwarded to the closed channel {}",
													&htlc.payment_hash, &monitor.channel_id());
												false
											} else { true }
										} else { true }
									});
									!forwards.is_empty()
								});
								pending_intercepted_htlcs.as_mut().unwrap().retain(|intercepted_id, htlc_info| {
									if pending_forward_matches_htlc(&htlc_info) {
										log_info!(logger, "Removing pending intercepted HTLC with hash {} as it was forwarded to the closed channel {}",
											&htlc.payment_hash, &monitor.channel_id());
										pending_events_read.retain(|(event, _)| {
											if let Event::HTLCIntercepted { intercept_id: ev_id, .. } = event {
												intercepted_id != ev_id
											} else { true }
										});
										false
									} else { true }
								});
							},
							HTLCSource::OutboundRoute {
								payment_id, session_priv, path, ..
							} => {
								if let Some(preimage) = preimage_opt {
									let pending_events = Mutex::new(pending_events_read);
									// Note that we set `from_onchain` to "false" here,
									// deliberately keeping the pending payment around forever.
									// Given it should only occur when we have a channel we're
									// force-closing for being stale that's okay.
									// The alternative would be to wipe the state when claiming,
									// generating a `PaymentPathSuccessful` event but regenerating
									// it and the `PaymentSent` on every restart until the
									// `ChannelMonitor` is removed.
									let compl_action =
										EventCompletionAction::ReleaseRAAChannelMonitorUpdate {
											channel_funding_outpoint: monitor.get_funding_txo().0,
											channel_id: monitor.channel_id(),
											counterparty_node_id: path.hops[0].pubkey,
										};
									pending_outbounds.claim_htlc(
										payment_id,
										preimage,
										session_priv,
										path,
										false,
										compl_action,
										&pending_events,
										&&logger,
									);
									pending_events_read = pending_events.into_inner().unwrap();
								}
							},
						}
					}
				}

				// Whether the downstream channel was closed or not, try to re-apply any payment
				// preimages from it which may be needed in upstream channels for forwarded
				// payments.
				let outbound_claimed_htlcs_iter = monitor
					.get_all_current_outbound_htlcs()
					.into_iter()
					.filter_map(|(htlc_source, (htlc, preimage_opt))| {
						if let HTLCSource::PreviousHopData(_) = htlc_source {
							if let Some(payment_preimage) = preimage_opt {
								Some((
									htlc_source,
									payment_preimage,
									htlc.amount_msat,
									htlc.amount_rgb,
									// Check if `counterparty_opt.is_none()` to see if the
									// downstream chan is closed (because we don't have a
									// channel_id -> peer map entry).
									counterparty_opt.is_none(),
									counterparty_opt
										.cloned()
										.or(monitor.get_counterparty_node_id()),
									monitor.get_funding_txo().0,
									monitor.channel_id(),
								))
							} else {
								None
							}
						} else {
							// If it was an outbound payment, we've handled it above - if a preimage
							// came in and we persisted the `ChannelManager` we either handled it and
							// are good to go or the channel force-closed - we don't have to handle the
							// channel still live case here.
							None
						}
					});
				for tuple in outbound_claimed_htlcs_iter {
					pending_claims_to_replay.push(tuple);
				}
			}
		}

		if !forward_htlcs.is_empty()
			|| !decode_update_add_htlcs.is_empty()
			|| pending_outbounds.needs_abandon()
		{
			// If we have pending HTLCs to forward, assume we either dropped a
			// `PendingHTLCsForwardable` or the user received it but never processed it as they
			// shut down before the timer hit. Either way, set the time_forwardable to a small
			// constant as enough time has likely passed that we should simply handle the forwards
			// now, or at least after the user gets a chance to reconnect to our peers.
			pending_events_read.push_back((
				events::Event::PendingHTLCsForwardable { time_forwardable: Duration::from_secs(2) },
				None,
			));
		}

		let inbound_pmt_key_material = args.node_signer.get_inbound_payment_key_material();
		let expanded_inbound_key = inbound_payment::ExpandedKey::new(&inbound_pmt_key_material);

		let mut claimable_payments = hash_map_with_capacity(claimable_htlcs_list.len());
		if let Some(purposes) = claimable_htlc_purposes {
			if purposes.len() != claimable_htlcs_list.len() {
				return Err(DecodeError::InvalidValue);
			}
			if let Some(onion_fields) = claimable_htlc_onion_fields {
				if onion_fields.len() != claimable_htlcs_list.len() {
					return Err(DecodeError::InvalidValue);
				}
				for (purpose, (onion, (payment_hash, htlcs))) in purposes
					.into_iter()
					.zip(onion_fields.into_iter().zip(claimable_htlcs_list.into_iter()))
				{
					let existing_payment = claimable_payments.insert(
						payment_hash,
						ClaimablePayment { purpose, htlcs, onion_fields: onion },
					);
					if existing_payment.is_some() {
						return Err(DecodeError::InvalidValue);
					}
				}
			} else {
				for (purpose, (payment_hash, htlcs)) in
					purposes.into_iter().zip(claimable_htlcs_list.into_iter())
				{
					let existing_payment = claimable_payments.insert(
						payment_hash,
						ClaimablePayment { purpose, htlcs, onion_fields: None },
					);
					if existing_payment.is_some() {
						return Err(DecodeError::InvalidValue);
					}
				}
			}
		} else {
			// LDK versions prior to 0.0.107 did not write a `pending_htlc_purposes`, but do
			// include a `_legacy_hop_data` in the `OnionPayload`.
			for (payment_hash, htlcs) in claimable_htlcs_list.drain(..) {
				if htlcs.is_empty() {
					return Err(DecodeError::InvalidValue);
				}
				let purpose = match &htlcs[0].onion_payload {
					OnionPayload::Invoice { _legacy_hop_data } => {
						if let Some(hop_data) = _legacy_hop_data {
							events::PaymentPurpose::Bolt11InvoicePayment {
								payment_preimage: match pending_inbound_payments.get(&payment_hash)
								{
									Some(inbound_payment) => inbound_payment.payment_preimage,
									None => match inbound_payment::verify(
										payment_hash,
										&hop_data,
										0,
										&expanded_inbound_key,
										&args.logger,
									) {
										Ok((payment_preimage, _)) => payment_preimage,
										Err(()) => {
											log_error!(args.logger, "Failed to read claimable payment data for HTLC with payment hash {} - was not a pending inbound payment and didn't match our payment key", &payment_hash);
											return Err(DecodeError::InvalidValue);
										},
									},
								},
								payment_secret: hop_data.payment_secret,
							}
						} else {
							return Err(DecodeError::InvalidValue);
						}
					},
					OnionPayload::Spontaneous(payment_preimage) => {
						events::PaymentPurpose::SpontaneousPayment(*payment_preimage)
					},
				};
				claimable_payments
					.insert(payment_hash, ClaimablePayment { purpose, htlcs, onion_fields: None });
			}
		}

		let mut secp_ctx = Secp256k1::new();
		secp_ctx.seeded_randomize(&args.entropy_source.get_secure_random_bytes());

		let our_network_pubkey = match args.node_signer.get_node_id(Recipient::Node) {
			Ok(key) => key,
			Err(()) => return Err(DecodeError::InvalidValue),
		};
		if let Some(network_pubkey) = received_network_pubkey {
			if network_pubkey != our_network_pubkey {
				log_error!(
					args.logger,
					"Key that was generated does not match the existing key. left: {}, right: {}",
					network_pubkey,
					our_network_pubkey
				);
				return Err(DecodeError::InvalidValue);
			}
		}

		let mut outbound_scid_aliases = new_hash_set();
		for (_peer_node_id, peer_state_mutex) in per_peer_state.iter_mut() {
			let mut peer_state_lock = peer_state_mutex.lock().unwrap();
			let peer_state = &mut *peer_state_lock;
			for (chan_id, phase) in peer_state.channel_by_id.iter_mut() {
				if let ChannelPhase::Funded(chan) = phase {
					let logger = WithChannelContext::from(&args.logger, &chan.context);
					if chan.context.outbound_scid_alias() == 0 {
						let mut outbound_scid_alias;
						loop {
							outbound_scid_alias = fake_scid::Namespace::OutboundAlias
								.get_fake_scid(
									best_block_height,
									&chain_hash,
									fake_scid_rand_bytes.as_ref().unwrap(),
									&args.entropy_source,
								);
							if outbound_scid_aliases.insert(outbound_scid_alias) {
								break;
							}
						}
						chan.context.set_outbound_scid_alias(outbound_scid_alias);
					} else if !outbound_scid_aliases.insert(chan.context.outbound_scid_alias()) {
						// Note that in rare cases its possible to hit this while reading an older
						// channel if we just happened to pick a colliding outbound alias above.
						log_error!(
							logger,
							"Got duplicate outbound SCID alias; {}",
							chan.context.outbound_scid_alias()
						);
						return Err(DecodeError::InvalidValue);
					}
					if chan.context.is_usable() {
						if short_to_chan_info
							.insert(
								chan.context.outbound_scid_alias(),
								(chan.context.get_counterparty_node_id(), *chan_id),
							)
							.is_some()
						{
							// Note that in rare cases its possible to hit this while reading an older
							// channel if we just happened to pick a colliding outbound alias above.
							log_error!(
								logger,
								"Got duplicate outbound SCID alias; {}",
								chan.context.outbound_scid_alias()
							);
							return Err(DecodeError::InvalidValue);
						}
					}
				} else {
					// We shouldn't have persisted (or read) any unfunded channel types so none should have been
					// created in this `channel_by_id` map.
					debug_assert!(false);
					return Err(DecodeError::InvalidValue);
				}
			}
		}

		let bounded_fee_estimator = LowerBoundedFeeEstimator::new(args.fee_estimator);

		for (_, monitor) in args.channel_monitors.iter() {
			for (payment_hash, payment_preimage) in monitor.get_stored_preimages() {
				if let Some(payment) = claimable_payments.remove(&payment_hash) {
					log_info!(args.logger, "Re-claiming HTLCs with payment hash {} as we've released the preimage to a ChannelMonitor!", &payment_hash);
					let mut claimable_amt_msat = 0;
					let mut receiver_node_id = Some(our_network_pubkey);
					let phantom_shared_secret = payment.htlcs[0].prev_hop.phantom_shared_secret;
					if phantom_shared_secret.is_some() {
						let phantom_pubkey = args
							.node_signer
							.get_node_id(Recipient::PhantomNode)
							.expect("Failed to get node_id for phantom node recipient");
						receiver_node_id = Some(phantom_pubkey)
					}
					for claimable_htlc in &payment.htlcs {
						claimable_amt_msat += claimable_htlc.value;

						// Add a holding-cell claim of the payment to the Channel, which should be
						// applied ~immediately on peer reconnection. Because it won't generate a
						// new commitment transaction we can just provide the payment preimage to
						// the corresponding ChannelMonitor and nothing else.
						//
						// We do so directly instead of via the normal ChannelMonitor update
						// procedure as the ChainMonitor hasn't yet been initialized, implying
						// we're not allowed to call it directly yet. Further, we do the update
						// without incrementing the ChannelMonitor update ID as there isn't any
						// reason to.
						// If we were to generate a new ChannelMonitor update ID here and then
						// crash before the user finishes block connect we'd end up force-closing
						// this channel as well. On the flip side, there's no harm in restarting
						// without the new monitor persisted - we'll end up right back here on
						// restart.
						let previous_channel_id = claimable_htlc.prev_hop.channel_id;
						if let Some(peer_node_id) =
							outpoint_to_peer.get(&claimable_htlc.prev_hop.outpoint)
						{
							let peer_state_mutex = per_peer_state.get(peer_node_id).unwrap();
							let mut peer_state_lock = peer_state_mutex.lock().unwrap();
							let peer_state = &mut *peer_state_lock;
							if let Some(ChannelPhase::Funded(channel)) =
								peer_state.channel_by_id.get_mut(&previous_channel_id)
							{
								let logger =
									WithChannelContext::from(&args.logger, &channel.context);
								channel.claim_htlc_while_disconnected_dropping_mon_update(
									claimable_htlc.prev_hop.htlc_id,
									payment_preimage,
									&&logger,
								);
							}
						}
						if let Some(previous_hop_monitor) =
							args.channel_monitors.get(&claimable_htlc.prev_hop.outpoint)
						{
							previous_hop_monitor.provide_payment_preimage(
								&payment_hash,
								&payment_preimage,
								&args.tx_broadcaster,
								&bounded_fee_estimator,
								&args.logger,
							);
						}
					}
					pending_events_read.push_back((
						events::Event::PaymentClaimed {
							receiver_node_id,
							payment_hash,
							purpose: payment.purpose,
							amount_msat: claimable_amt_msat,
							htlcs: payment.htlcs.iter().map(events::ClaimedHTLC::from).collect(),
							sender_intended_total_msat: payment
								.htlcs
								.first()
								.map(|htlc| htlc.total_msat),
						},
						None,
					));
				}
			}
		}

		for (node_id, monitor_update_blocked_actions) in
			monitor_update_blocked_actions_per_peer.unwrap()
		{
			if let Some(peer_state) = per_peer_state.get(&node_id) {
				for (channel_id, actions) in monitor_update_blocked_actions.iter() {
					let logger = WithContext::from(&args.logger, Some(node_id), Some(*channel_id));
					for action in actions.iter() {
						if let MonitorUpdateCompletionAction::EmitEventAndFreeOtherChannel {
							downstream_counterparty_and_funding_outpoint:
								Some((
									blocked_node_id,
									_blocked_channel_outpoint,
									blocked_channel_id,
									blocking_action,
								)),
							..
						} = action
						{
							if let Some(blocked_peer_state) = per_peer_state.get(blocked_node_id) {
								log_trace!(logger,
									"Holding the next revoke_and_ack from {} until the preimage is durably persisted in the inbound edge's ChannelMonitor",
									blocked_channel_id);
								blocked_peer_state
									.lock()
									.unwrap()
									.actions_blocking_raa_monitor_updates
									.entry(*blocked_channel_id)
									.or_insert_with(Vec::new)
									.push(blocking_action.clone());
							} else {
								// If the channel we were blocking has closed, we don't need to
								// worry about it - the blocked monitor update should never have
								// been released from the `Channel` object so it can't have
								// completed, and if the channel closed there's no reason to bother
								// anymore.
							}
						}
						if let MonitorUpdateCompletionAction::FreeOtherChannelImmediately {
							..
						} = action
						{
							debug_assert!(false, "Non-event-generating channel freeing should not appear in our queue");
						}
					}
				}
				peer_state.lock().unwrap().monitor_update_blocked_actions =
					monitor_update_blocked_actions;
			} else {
				log_error!(
					WithContext::from(&args.logger, Some(node_id), None),
					"Got blocked actions without a per-peer-state for {}",
					node_id
				);
				return Err(DecodeError::InvalidValue);
			}
		}

		let channel_manager = ChannelManager {
			chain_hash,
			fee_estimator: bounded_fee_estimator,
			chain_monitor: args.chain_monitor,
			tx_broadcaster: args.tx_broadcaster,
			router: args.router,

			best_block: RwLock::new(BestBlock::new(best_block_hash, best_block_height)),

			inbound_payment_key: expanded_inbound_key,
			pending_inbound_payments: Mutex::new(pending_inbound_payments),
			pending_outbound_payments: pending_outbounds,
			pending_intercepted_htlcs: Mutex::new(pending_intercepted_htlcs.unwrap()),

			forward_htlcs: Mutex::new(forward_htlcs),
			decode_update_add_htlcs: Mutex::new(decode_update_add_htlcs),
			claimable_payments: Mutex::new(ClaimablePayments {
				claimable_payments,
				pending_claiming_payments: pending_claiming_payments.unwrap(),
			}),
			outbound_scid_aliases: Mutex::new(outbound_scid_aliases),
			outpoint_to_peer: Mutex::new(outpoint_to_peer),
			short_to_chan_info: FairRwLock::new(short_to_chan_info),
			fake_scid_rand_bytes: fake_scid_rand_bytes.unwrap(),

			probing_cookie_secret: probing_cookie_secret.unwrap(),

			our_network_pubkey,
			secp_ctx,

			highest_seen_timestamp: AtomicUsize::new(highest_seen_timestamp as usize),

			per_peer_state: FairRwLock::new(per_peer_state),

			pending_events: Mutex::new(pending_events_read),
			pending_events_processor: AtomicBool::new(false),
			pending_background_events: Mutex::new(pending_background_events),
			total_consistency_lock: RwLock::new(()),
			background_events_processed_since_startup: AtomicBool::new(false),

			event_persist_notifier: Notifier::new(),
			needs_persist_flag: AtomicBool::new(false),

			funding_batch_states: Mutex::new(BTreeMap::new()),

			pending_offers_messages: Mutex::new(Vec::new()),

			pending_broadcast_messages: Mutex::new(Vec::new()),

			entropy_source: args.entropy_source,
			node_signer: args.node_signer,
			signer_provider: args.signer_provider,

			logger: args.logger,
			default_configuration: args.default_config,
			color_source: args.color_source,
		};

		for htlc_source in failed_htlcs.drain(..) {
			let (source, payment_hash, counterparty_node_id, channel_id) = htlc_source;
			let receiver =
				HTLCDestination::NextHopChannel { node_id: Some(counterparty_node_id), channel_id };
			let reason = HTLCFailReason::from_failure_code(0x4000 | 8);
			channel_manager.fail_htlc_backwards_internal(&source, &payment_hash, &reason, receiver);
		}

		for (
			source,
			preimage,
			downstream_value,
			downstream_rgb,
			downstream_closed,
			downstream_node_id,
			downstream_funding,
			downstream_channel_id,
		) in pending_claims_to_replay
		{
			// We use `downstream_closed` in place of `from_onchain` here just as a guess - we
			// don't remember in the `ChannelMonitor` where we got a preimage from, but if the
			// channel is closed we just assume that it probably came from an on-chain claim.
			channel_manager.claim_funds_internal(
				source,
				preimage,
				Some(downstream_value),
				downstream_rgb,
				None,
				downstream_closed,
				true,
				downstream_node_id,
				downstream_funding,
				downstream_channel_id,
				None,
			);
		}

		//TODO: Broadcast channel update for closed channels, but only after we've made a
		//connection or two.

		Ok((best_block_hash.clone(), channel_manager))
	}
}
```

## 教学

下面我告诉你怎么读。

首先，我根本不读！我直接选中 `Readable::read` 然后按下 `Ctrl+Shift+L`，这在我的编辑器中表示选中所有的相同文本。然后我进而选中整行。

考虑到还有一些反序列化位于循环语句中，再手动浏览一遍将其补上。得到下面的代码：

```rust
		let chain_hash: ChainHash = Readable::read(reader)?;
		let best_block_height: u32 = Readable::read(reader)?;
		let best_block_hash: BlockHash = Readable::read(reader)?;
		let channel_count: u64 = Readable::read(reader)?;
		let forward_htlcs_count: u64 = Readable::read(reader)?;
		for _ in 0..forward_htlcs_count {
			let short_channel_id = Readable::read(reader)?;
			let pending_forwards_count: u64 = Readable::read(reader)?;
				pending_forwards.push(Readable::read(reader)?);
		}
		let claimable_htlcs_count: u64 = Readable::read(reader)?;
		for _ in 0..claimable_htlcs_count {
			let payment_hash = Readable::read(reader)?;
			let previous_hops_len: u64 = Readable::read(reader)?;
		}
		let peer_count: u64 = Readable::read(reader)?;
		for _ in 0..peer_count {
			let peer_pubkey = Readable::read(reader)?;
			peer_state.latest_features = Readable::read(reader)?;
		}
		let event_count: u64 = Readable::read(reader)?;
		for _ in 0..event_count {
			match MaybeReadable::read(reader)? {
		}
		let background_event_count: u64 = Readable::read(reader)?;
		for _ in 0..background_event_count {
					let _: OutPoint = Readable::read(reader)?;
					let _: ChannelMonitorUpdate = Readable::read(reader)?;
		}
		let _last_node_announcement_serial: u32 = Readable::read(reader)?; // Only used < 0.0.111
		let highest_seen_timestamp: u32 = Readable::read(reader)?;
		let pending_inbound_payment_count: u64 = Readable::read(reader)?;
		for _ in 0..pending_inbound_payment_count {
				.insert(Readable::read(reader)?, Readable::read(reader)?)
		}
		let pending_outbound_payments_count_compat: u64 = Readable::read(reader)?;
		for _ in 0..pending_outbound_payments_count_compat {
			let session_priv = Readable::read(reader)?;
		}
```

然后，这样做可能有漏网之鱼，再搜索 `reader` 找到遗漏的：

```rust
		let _ver = read_ver_prefix!(reader, SERIALIZATION_VERSION);
		for _ in 0..channel_count {
			let mut channel: Channel<SP> = Channel::read(
				reader,
				(
					&args.entropy_source,
					&args.signer_provider,
					best_block_height,
					&provided_channel_type_features(&args.default_config),
					args.color_source.clone(),
				),
			)?;
		}
		for _ in 0..claimable_htlcs_count {
			for _ in 0..previous_hops_len {
				previous_hops.push(<ClaimableHTLC as Readable>::read(reader)?);
			}
		}
		for _ in 0..background_event_count {
			match <u8 as Readable>::read(reader)? {
				0 => {
					// LDK versions prior to 0.0.116 wrote pending `MonitorUpdateRegeneratedOnStartup`s here,
					// however we really don't (and never did) need them - we regenerate all
					// on-startup monitor updates.
				},
				_ => return Err(DecodeError::InvalidValue),
			}
		}
		read_tlv_fields!(reader, {
			(1, pending_outbound_payments_no_retry, option),
			(2, pending_intercepted_htlcs, option),
			(3, pending_outbound_payments, option),
			(4, pending_claiming_payments, option),
			(5, received_network_pubkey, option),
			(6, monitor_update_blocked_actions_per_peer, option),
			(7, fake_scid_rand_bytes, option),
			(8, events_override, option),
			(9, claimable_htlc_purposes, optional_vec),
			(10, in_flight_monitor_updates, option),
			(11, probing_cookie_secret, option),
			(13, claimable_htlc_onion_fields, optional_vec),
			(14, decode_update_add_htlcs, option),
		});
```

好的，齐活了。

## 最终成果


| **序号** | **字段**                                   | **类型**                             | **描述**                                                                 |
|-------|-----------------------------------------|--------------------------------|---------------------------------------------------------------------|
| 1     | `chain_hash`                            | `ChainHash`                   | 区块链网络的标识符。                                                        |
| 2     | `best_block_height`                     | `u32`                         | 最优（最近）区块的高度。                                                    |
| 3     | `best_block_hash`                       | `BlockHash`                   | 最优区块的哈希值。                                                          |
| 4     | `channel_count`                         | `u64`                         | 管理的通道总数。                                                            |
| 5     | `forward_htlcs_count`                   | `u64`                         | 转发 HTLC 的数量。                                                         |
| 6     | **转发 HTLCs**                           |                                |                                                                     |
|       | &nbsp;&nbsp; `short_channel_id`         | `ShortChannelId`              | 短通道的标识符。                                                            |
|       | &nbsp;&nbsp; `pending_forwards_count`   | `u64`                         | 当前通道中挂起转发 HTLC 的数量。                                              |
|       | &nbsp;&nbsp; `pending_forwards`         | `PendingForward` (列表)         | 挂起转发 HTLC 的列表。                                                      |
| 7     | `claimable_htlcs_count`                 | `u64`                         | 可领取的 HTLC 数量。                                                        |
| 8     | **可领取的 HTLCs**                       |                                |                                                                     |
|       | &nbsp;&nbsp; `payment_hash`             | `PaymentHash`                 | 与 HTLC 相关的支付哈希。                                                     |
|       | &nbsp;&nbsp; `previous_hops_len`        | `u64`                         | HTLC 路径中的前一跳数量。                                                   |
| 9     | `peer_count`                            | `u64`                         | 连接的节点总数。                                                            |
| 10    | **节点**                                 |                                |                                                                     |
|       | &nbsp;&nbsp; `peer_pubkey`              | `PubKey`                      | 节点的公钥。                                                                |
|       | &nbsp;&nbsp; `peer_state.latest_features` | `LatestFeatures`             | 节点支持的最新功能集。                                                       |
| 11    | `event_count`                           | `u64`                         | 记录的事件数量。                                                            |
| 12    | **事件**                                 |                                | 使用 `MaybeReadable` 反序列化的事件列表。                                     |
| 13    | `background_event_count`                | `u64`                         | 后台事件的数量。                                                            |
| 14    | **后台事件**                              |                                |                                                                     |
|       | &nbsp;&nbsp; `OutPoint`                 | `OutPoint`                   | 引用特定 UTXO 的输入。                                                       |
|       | &nbsp;&nbsp; `ChannelMonitorUpdate`     | `ChannelMonitorUpdate`        | 通道监控的更新信息。                                                        |
| 15    | `_last_node_announcement_serial`        | `u32`                         | （已弃用）最后一个节点公告的序列号（仅适用于版本 < 0.0.111）。                        |
| 16    | `highest_seen_timestamp`                | `u32`                         | 最近一次事件的时间戳。                                                      |
| 17    | `pending_inbound_payment_count`         | `u64`                         | 挂起的入站支付数量。                                                        |
| 18    | **挂起的入站支付**                        |                                |                                                                     |
|       | &nbsp;&nbsp; `PaymentId`                | `PaymentId`                  | 入站支付的标识符。                                                          |
|       | &nbsp;&nbsp; `PaymentData`              | `PaymentData`                | 与挂起入站支付相关的数据。                                                   |
| 19    | `pending_outbound_payments_count_compat` | `u64`                        | 挂起的出站支付数量（兼容层）。                                                |
| 20    | **挂起的出站支付（兼容层）**               |                                |                                                                     |
|       | &nbsp;&nbsp; `session_priv`             | `SessionPriv`                | 出站支付的会话私有数据。                                                     |
| 21    | `_ver`                                  | `u8` (版本前缀)                 | 序列化的版本前缀（与 `SERIALIZATION_VERSION` 验证）。                           |
| 22    | **通道**                                 |                                |                                                                     |
|       | &nbsp;&nbsp; `Channel<SP>`              | `Channel` (列表)                | 通道状态的列表，通过上下文参数反序列化。                                        |
| 23    | **可领取 HTLC 的前一跳**                  |                                |                                                                     |
|       | &nbsp;&nbsp; `ClaimableHTLC`            | `ClaimableHTLC` (列表)          | 每个可领取 HTLC 的前一跳详细信息。                                             |
| 24    | **后台事件验证**                          |                                |                                                                     |
|       | &nbsp;&nbsp; `u8`                       | `u8`                         | 验证后台事件，仅接受 `0`（LDK < 0.0.116）。                                 |
| 25    | **TLV 字段**                              |                                |                                                                     |
|       | &nbsp;&nbsp; `1: pending_outbound_payments_no_retry` | 可选字段                     | 无重试的挂起出站支付。                                                       |
|       | &nbsp;&nbsp; `2: pending_intercepted_htlcs` | 可选字段                     | 挂起的拦截 HTLCs。                                                          |
|       | &nbsp;&nbsp; `3: pending_outbound_payments` | 可选字段                     | 可重试的挂起出站支付。                                                       |
|       | &nbsp;&nbsp; `4: pending_claiming_payments` | 可选字段                     | 可领取的挂起支付。                                                           |
|       | &nbsp;&nbsp; `5: received_network_pubkey` | 可选字段                     | 从节点接收到的网络公钥。                                                      |
|       | &nbsp;&nbsp; `6: monitor_update_blocked_actions_per_peer` | 可选字段               | 每个节点阻止的监控更新操作。                                                  |
|       | &nbsp;&nbsp; `7: fake_scid_rand_bytes` | 可选字段                     | 假短通道 ID (SCID) 的随机字节。                                               |
|       | &nbsp;&nbsp; `8: events_override`      | 可选字段                     | 事件处理的重写选项。                                                         |
|       | &nbsp;&nbsp; `9: claimable_htlc_purposes` | 可选向量字段                | 与可领取 HTLC 相关的用途。                                                   |
|       | &nbsp;&nbsp; `10: in_flight_monitor_updates` | 可选字段                  | 当前正在进行的监控更新。                                                     |
|       | &nbsp;&nbsp; `11: probing_cookie_secret` | 可选字段                   | 用于探测 cookie 的密钥。                                                     |
|       | &nbsp;&nbsp; `13: claimable_htlc_onion_fields` | 可选向量字段              | 可领取 HTLC 的洋葱包字段。                                                   |
|       | &nbsp;&nbsp; `14: decode_update_add_htlcs` | 可选字段                   | 添加 HTLC 的更新解码。                                                       |

## 总结

效率！效率！还是效率！

当阅读这种业务复杂、逻辑简单但又特别长的代码时，我们核心点在于不要试图死磕，而是着力于函数各个输入具体被怎样操作，产生怎样的输出或 side effect。这样可以有效提高阅读效率，达成阅读的目标。在这个过程中，你可以自行探索各种技巧（欢迎 PR 补充！）。
