<template>
  <section class="autopilot-settings">
    <div class="slideout-header">
      <a @click.prevent="closePanel">
        <span class="icon is-small">
          <font-awesome-icon :icon="['fas', 'chevron-left']"/>
        </span>

        <span>Back</span>
      </a>

      <UnitSwitch />
    </div>

    <div class="app-slideout lnd channel">
      <div class="app-title">
        <img src="~assets/lightning.png" alt="">
        <h2>
          Manage Autopilot
        </h2>
      </div>

      <div class="autopilot-toggle">
        <div class="field light-grey">
          Autopilot
          <span v-if="system.settings.lnd.autopilot">On</span>
          <span v-else>Off</span>
        </div>
        <div class="field">
          <b-switch v-model="system.settings.lnd.autopilot" type="is-info" size="is-medium" @input="toggleAutopilot"></b-switch>
        </div>
      </div>

      <p class="description">
        Autopilot tries its best to match your settings, but may not always open channels to the exact size you specify.
        Occasionally, your channels may be opened at lower amounts based on other nodes' availability.
      </p>

      <hr>

      <template v-if="system.settings.lnd.autopilot">
        <!-- Lightning Stats Section -->
        <div class="stats">
          <div class="stats-col">
            <h1>{{openAutopilot.length}} Channel<template v-if="openAutopilot.length != 1">s</template></h1>
            <h2>{{balanceTotal | inUnits | withSuffix}} in Autopilot</h2>
          </div>
          <div class="stats-col">
            <h1>{{pendingAutopilot.length}} Pending</h1>
            <h2>{{balancePending | inUnits | withSuffix}} Pending</h2>
          </div>
          <div class="stats-col">
            <h1>{{autopilotAverage | inUnits | withSuffix}}</h1>
            <h2>Avg. Value per Channel</h2>
          </div>
        </div>
        <hr>

        <div class="field toggle-settings menu-navigation is-horizontal">
          <div class="field-label is-normal">
            <label class="label">
              Autopilot Settings
            </label>
            <p>Allocate funds in your Autopilot channels.</p>
          </div>

          <div class="field-body">
            <div class="field">
              <a class="button is-rounded" @click="openSettings">
                <span>Configure</span>
                <span class="icon is-small">
                  <font-awesome-icon :icon="['fas', 'chevron-right']"/>
                </span>
              </a>
            </div>
          </div>
        </div>
        <hr>
      </template>

      <template v-if="channelList.length">
        <section class="transaction-settings">
          <div class="section-title">Autopilot Channels</div>

          <!-- Transactions -->
          <div class="tx-list">
            <ul class="pending-tx">
              <li class="tx-item" v-for="(ch, index) in channelList" :key="index">
                <div class="tx-row" @click="manageChannel(ch)">
                  <div class="tx-col-1 desktop-only">
                    <div class="channel-icon">
                      <img src="~assets/channel.svg" alt="">
                    </div>
                  </div>
                  <div class="tx-col-2">
                    <h2>Autopilot Channel</h2>
                    <h3>
                      <a href="#">{{ch.remotePubkey}}</a>
                    </h3>
                  </div>
                  <div class="tx-col-3">
                    <!-- TODO: handle active color with css-->
                    <h2 class="channel-status " v-bind:style="{color: ch.activeColor}">{{ch.status}}</h2>
                    <h3 class="channel-balance" v-if="ch.timeRemainingText">{{ch.timeRemainingText}}</h3>
                    <h3 class="channel-balance" v-if="!ch.timeRemainingText">{{ch.localBalance | inUnits | withSuffix}}</h3>
                  </div>
                </div>
                <hr>
              </li>
            </ul>
          </div>
        </section>
      </template> <!-- End of autopilot data -->

      <template v-else>
        <div class="inactive-wrap">
          <div class="inactive">
            <span class="icon is-large">
              <img src="~assets/channel.svg" alt="">
            </span>

            <h2 v-if="system.settings.lnd.autopilot">Configure your settings so Autopilot can start.</h2>
            <h2 v-else>Autopilot is currently inactive.</h2>
          </div>
        </div>
      </template>

      </div>

  </section>
</template>

<script>
import axios from 'axios';
import EventBus from '@/helpers/event-bus';
import LightningData from '@/data/lightning';
import SystemData from '@/data/system';
import {satsToBtc, btcToSats} from '@/helpers/units';

import ManageChannel from '@/components/Lightning/Modals/Channels/ManageChannel';
import AutopilotSettings from '@/components/Lightning/Modals/AutopilotSettings';
import UnitSwitch from '@/components/Settings/UnitSwitch';

export default {
  name: 'Autopilot',

  async created() {
    EventBus.$emit('load-lightning-stats');
    this.setChannelData();
  },

  mounted() {
    var vm = this; // avoid scope problems
    EventBus.$on('cancel', () => vm.closePanel());
  },

  data() {
    return {
      lightning: LightningData,
      system: SystemData,
      channelList: [],
      balanceTotal: 0,
      balancePending: 0,
    }
  },

  destroyed() {
    EventBus.$emit('stop-lightning-stats');
    EventBus.$off('cancel');
  },

  computed: {
    autopilotAverage() {
      // If there are any open channels, determine average balance per channel
      const open = [];

      this.lightning.channels.forEach((channel) => {
        if(!channel.managed && channel.initiator) {
          if(channel.status === 'open') {
            open.push(channel);
          }
        }
      });

      if(open.length) {
        return this.lightning.balance.confirmed / open.length;
      }

      return 0;
    },

    openAutopilot() {
      const open = [];

      this.lightning.channels.forEach((channel) => {
        if(!channel.managed && channel.initiator) {
          if(channel.status === 'open') {
            open.push(channel);
          }
        }
      });

      return open;
    },

    pendingAutopilot() {
      const pending = [];

      this.lightning.channels.forEach((channel) => {
        if(!channel.managed && channel.initiator) {
          if(channel.status === 'pending') {
            pending.push(channel);
          }
        }
      });

      return pending;
    },
  },

  methods: {
    closePanel() {
      this.$emit('closePanel');
      this.$destroy();
    },
    manageChannel(data) {
      data.localBalanceBtc = satsToBtc(data.localBalance);
      data.remoteBalanceBtc = satsToBtc(data.remoteBalance);

      // Number of blocks * average block time in minutes / minutes per hour
      if (data.csvDelay) {
        data.csvDelayText = this.blocksToTime(data.csvDelay);
      }

      // Denote that this is an autopilot channel
      data.autopilot = true;
      data.name = 'Autopilot';
      data.purpose = 'Autopilot';

      this.$modal.open({
        parent: this,
        props: {chData: data},
        component: ManageChannel,
        hasModalCard: true
      })
    },
    blocksToTime(blocks) {
      // TODO: find a library for this
      const minutes = (blocks * 10);

      if (minutes <= 50) {
        return '~' + minutes + ' minutes';
      }

      const hours = minutes / 60;

      if (hours === 1) {
        return '~1 hour';
      }
      return '~' + hours.toFixed(0) + ' hours';
    },
    openSettings() {
      this.$modal.open({
        parent: this,
        component: AutopilotSettings,
        hasModalCard: true,
      })
    },
    channelPending() {
      this.$toast.open({duration: 3000, message:`New Channel Pending`});
      this.setChannelData();
    },
    channelClosing() {
      this.$toast.open({duration: 3000, message:`New Channel Closing`});
      this.setChannelData();
    },
    async setChannelData() {
      const channels = (await this.$axios.get(`${this.$env.API_LND}/v1/lnd/channel`)).data;

      // The determines the order
      this.channelList = [];

      for (const channel of channels) {

        // We should ignore channels not created by autopilot
        if (channel.managed || !channel.initiator) {
          continue;
        }

        if (channel.type === 'OPEN') {
          if (channel.active) {
            channel.activeColor = '#2dcccd';
            channel.status = 'Online';

            this.balanceTotal += parseInt(channel.localBalance) || 0;
          } else {
            channel.activeColor = '#f0649e'; //  medium-pink
            channel.status = 'Offline';
            this.balancePending += parseInt(channel.localBalance) || 0;
          }
          // Closing channels that are unconfirmed
        } else if (channel.type === 'WAITING_CLOSING_CHANNEL') {
          channel.activeColor = '#f7bd00'; //  golden
          channel.status = 'Closing';
          this.balancePending += parseInt(channel.localBalance) || 0;

        // Non cooperative closes that have at least one confirmation
        } else if (channel.type === 'FORCE_CLOSING_CHANNEL') {
          channel.activeColor = '#f7bd00'; //  golden
          channel.status = 'Closing';
          this.balancePending += parseInt(channel.localBalance) || 0;

        // Cooperative closes that have at least one confirmation
        } else if (channel.type === 'PENDING_CLOSING_CHANNEL') {
          channel.activeColor = '#f7bd00'; //  golden
          channel.status = 'Closing';
          this.balancePending += parseInt(channel.localBalance) || 0;

        // Pending open channels could be confirmed or unconfirmed
        } else if (channel.type === 'PENDING_OPEN_CHANNEL') {
          channel.activeColor = '#f7bd00'; //  golden
          channel.status = 'Opening';
          this.balancePending += parseInt(channel.localBalance) || 0;
        }

        if (channel.remainingConfirmations) {
          channel.timeRemainingText = this.blocksToTime(channel.remainingConfirmations);
        }

        this.channelList.push(channel);
      }
    },

    toggleAutopilot() {
      const message = this.system.settings.lnd.autopilot ? 'Autopilot enabled' : 'Autopilot disabled';
      const data = {
        autopilot: this.system.settings.lnd.autopilot,
        maxChannels: parseInt(this.system.settings.lnd.maxChannels) || 0,
        maxChanSize: parseInt(this.system.settings.lnd.maxChanSize) || 0,
      };

      this.saveSettings(data, message);
    },

    async saveSettings(data, message = 'Settings Saved') {
      EventBus.$emit('loading-start');

      try {
        await this.$axios.post(`${this.$env.API_MANAGER}/v1/settings/save`, data);
        this.$toast.open({duration: 3000, message: message});
      } catch (error) {
        console.error(error);
      } finally {
        EventBus.$emit('loading-stop');
      }
    },
  },

  components: {
    UnitSwitch
  },
};
</script>
