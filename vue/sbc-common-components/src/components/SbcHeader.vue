<template>
  <header class="app-header">
    <div class="container">
      <a href="/cooperatives/" class="brand">
        <picture>
          <source media="(min-width: 601px)"
            srcset="../assets/img/gov_bc_logo_horiz.png">
          <source media="(max-width: 600px)"
            srcset="../assets/img/gov_bc_logo_vert.png">
          <img class="brand__image"
            src="../assets/img/gov_bc_logo_vert.png"
            alt="Government of British Columbia Logo"
            title="Government of British Columbia">
        </picture>
        <span class="brand__title">BC Registries <span class="brand__title--wrap">& Online Services</span></span>
      </a>
      <div class="app-header__actions">
        <v-btn color="#fcba19" class="log-in-btn" v-if="!authorized" @click="login">Log in with BC Services Card</v-btn>

        <!-- Messages -->
        <v-menu bottom left fixed transition="slide-y-transition" v-if="authorized">
          <template v-slot:activator="{ on }">
            <v-btn text large class="messages-btn mr-2" v-on="on">
              <v-icon class="white--text">
                mdi-bell-outline
              </v-icon>
              <v-badge dot overlap offset-y="-6" color="error" v-if="_pendingApprovalCount > 0"/>
              <v-icon small>mdi-chevron-down</v-icon>
            </v-btn>
          </template>
          <v-list tile dense>
            <!-- No Items -->
            <v-list-item v-if="_pendingApprovalCount === 0">
              <v-list-item-title>No actions required</v-list-item-title>
            </v-list-item>

            <v-list-item two-line v-if="_pendingApprovalCount > 0" @click="goToTeamManagement()">
              <v-list-item-content>
                <v-list-item-title>You have {{ _pendingApprovalCount }} pending approvals</v-list-item-title>
                <v-list-item-subtitle>{{ _pendingApprovalCount }} <span>{{_pendingApprovalCount == '1' ? 'team member' : 'team members'}}</span> require approval to access this account</v-list-item-subtitle>
              </v-list-item-content>
            </v-list-item>
          </v-list>
        </v-menu>

        <!-- Account -->
        <v-menu bottom left fixed transition="slide-y-transition" content-class="account-menu" v-if="authorized">
          <template v-slot:activator="{ on }">
            <v-btn text large v-on="on" class="user-account-btn">
              <v-avatar tile left size="32" class="user-avatar">
                {{ username.slice(0,1) }}
              </v-avatar>
              <div class="user-info">
                <div class="user-name" data-test="user-name">{{ username }}</div>
                <div class="account-name" v-if="accountType !== 'IDIR'" data-test="account-name">{{ _accountName }}</div>
              </div>
              <v-icon small class="ml-2">mdi-chevron-down</v-icon>
            </v-btn>
          </template>
          <v-list tile dense>
            <v-list-item two-line>
              <v-list-item-avatar tile size="36" color="#3f5c94" class="user-avatar mr-4">
                {{ username.slice(0,1) }}
              </v-list-item-avatar>
              <v-list-item-content class="user-info">
                <v-list-item-title class="user-name" data-test="menu-user-name">{{ username }}</v-list-item-title>
                <v-list-item-subtitle class="account-name" v-if="accountType !== 'IDIR'" data-test="menu-account-name">{{ _accountName }}</v-list-item-subtitle>
              </v-list-item-content>
            </v-list-item>
            <!-- BEGIN: Hide if authentication is IDIR -->
            <v-list-item @click="goToUserProfile" v-if="accountType !== 'IDIR'">
              <v-list-item-icon left>
                <v-icon>mdi-account-outline</v-icon>
              </v-list-item-icon>
              <v-list-item-title>Edit Profile</v-list-item-title>
            </v-list-item>
            <!-- END -->
            <v-list-item @click="logout">
              <v-list-item-icon left>
                <v-icon>mdi-logout-variant</v-icon>
              </v-list-item-icon>
              <v-list-item-title>Log out</v-list-item-title>
            </v-list-item>
          </v-list>

          <v-divider></v-divider>

          <v-list tile dense v-if="accountType !== 'IDIR'">
            <v-subheader>ACCOUNT SETTINGS</v-subheader>
            <v-list-item to="/account-settings/account-info">
              <v-list-item-icon left>
                <v-icon>mdi-information-outline</v-icon>
              </v-list-item-icon>
              <v-list-item-title>Account Info</v-list-item-title>
            </v-list-item>
            <v-list-item to="/account-settings/team-members">
              <v-list-item-icon left>
                <v-icon>mdi-account-group-outline</v-icon>
              </v-list-item-icon>
              <v-list-item-title>Team Members</v-list-item-title>
            </v-list-item>
            <v-list-item v-if="showAccountSwitching">HIDE ME</v-list-item>
          </v-list>
        </v-menu>
      </div>
    </div>
  </header>
</template>

<script lang="ts">
import { Component, Prop, Watch } from 'vue-property-decorator'
import Vue from 'vue'
import { integer } from 'vuelidate/lib/validators'
import { initialize, LDClient } from 'launchdarkly-js-client-sdk'
import ConfigHelper from '../util/config-helper'
import { SessionStorageKeys } from '../util/constants'

@Component({})
export default class SbcHeader extends Vue {
  @Prop({ default: '-' }) private accountName: string;
  @Prop({ default: 0 }) private pendingApprovalCount: number;
  private ldClient: LDClient;

  get showAccountSwitching (): boolean {
    try {
      const flags = JSON.parse(ConfigHelper.getFromSession(SessionStorageKeys.LaunchDarklyFlags))
      return flags && flags['account-switching']
    } catch (exception) {
      return false
    }
  }

  private get _accountName (): string {
    return this.accountName || ConfigHelper.getFromSession(SessionStorageKeys.AccountName) || '-'
  }

  private get _pendingApprovalCount (): number {
    try {
      return this.pendingApprovalCount ? this.pendingApprovalCount : parseInt(ConfigHelper.getFromSession(SessionStorageKeys.PendingApprovalCount) || '0')
    } catch (exception) {
      return 0
    }
  }

  @Watch('accountName')
  private onAccountNameChanged (val: string, oldVal: string): void {
    ConfigHelper.addToSession(SessionStorageKeys.AccountName, val)
  }

  @Watch('pendingApprovalCount')
  private onPendingApprovalCountChanged (val: number, oldVal: number): void {
    ConfigHelper.addToSession(SessionStorageKeys.PendingApprovalCount, val)
  }

  private get username () : string {
    return ConfigHelper.getFromSession(SessionStorageKeys.UserFullName) || '-'
  }

  private get authorized (): boolean {
    let auth = ConfigHelper.getFromSession(SessionStorageKeys.KeyCloakToken)
    return !!auth
  }

  private get accountType (): string {
    return ConfigHelper.getFromSession(SessionStorageKeys.UserAccountType) || 'BCSC'
  }

  private mounted () {
    // Initialize LaunchDarkly flags and sync to session storage
    const user = { 'key': 'sbc-common-components' }
    this.ldClient = initialize('5db9da115f58e008123cd783', user)
    this.ldClient.on('ready', () => {
      console.log('It\'s now safe to request feature flags')
      ConfigHelper.addToSession(SessionStorageKeys.LaunchDarklyFlags, JSON.stringify(this.ldClient.allFlags()))
    })
  }

  logout () {
    window.location.assign('/cooperatives/auth/signout')
  }

  login () {
    window.location.assign('/cooperatives/auth/signin/bcsc')
  }

  private goToUserProfile () {
    window.location.assign('/cooperatives/auth/userprofile')
  }

  private goToAccountSettings () {
    window.location.assign('/cooperatives/auth/accountsettings')
  }

  private goToTeamManagement () {
    window.location.assign('/cooperatives/auth/account-settings/team-members')
  }
}
</script>

<style lang="scss" scoped>
@import "../assets/scss/theme.scss";

$app-header-font-color: #ffffff;

.app-header {
  height: 70px;
  color: $app-header-font-color;
  border-bottom: 2px solid $BCgovGold5;
  background-color: #003366;

  .container {
    display: flex;
    align-items: center;
    height: 100%;
    padding-top: 0;
    padding-bottom: 0;
  }
}

.app-header__actions {
  display: flex;
  align-items: center;
  margin-left: auto;

  .v-btn {
    margin-right: 0;
  }
}

.brand {
  display: flex;
  align-items: center;
  padding-right: 1rem;
  text-decoration: none;
  color: inherit;
}

.brand__image {
  display: block;
  margin-right: 1.25rem;
  max-height: 70px;
}

.brand__title {
  letter-spacing: -0.03rem;
  font-size: 1.125rem;
  font-weight: 700;
  color: inherit;
}

@media (max-width: 600px) {
  .brand__image {
    margin-right: 0.75rem;
    margin-left: -0.15rem;
  }

  .brand__title {
    font-size: 1rem;
    line-height: 1.25rem;
  }

  .brand__title--wrap {
    display: block;
  }
}

.v-btn.user-account-btn {
  padding-right: 0.5rem !important;
  padding-left: 0.5rem !important;
  text-align: left;
  color: $app-header-font-color;
  letter-spacing: 0.02rem;
  font-size: 0.8rem;

  .user-avatar {
    margin-right: 0.75rem;
  }

  .user-name {
    line-height: 1.125rem;
    font-size: 0.75rem;
  }

  .account-name {
    margin-bottom: 0.01rem;
    font-size: 0.7rem;
    opacity: 0.75;
  }
}

.v-btn.messages-btn {
  min-width: auto !important;
  padding-right: 0.5rem !important;
  padding-left: 0.4rem !important;
  color: $app-header-font-color;

  .v-icon {
    font-size: 1.75rem;
  }

  .v-badge {
    margin-right: 0.25rem;
  }
}

@media (max-width: 960px) {
  .v-btn.user-account-btn {
    min-width: auto !important;
    font-size: 0.8rem;

    .user-avatar {
      margin-right: 0;
    }

    .user-info {
      display: none;
    }
  }
}

// Account Menu
.account-menu {
  background: #ffffff;
}

.account-menu__info {
  font-size: 0.875rem;
}

.v-list {
  border-radius: 0;

  .v-list-item__title,
  .v-list-item__subtitle {
    line-height: normal !important;
  }
}

.v-list .v-list-item__title.user-name,
.user-name {
  font-size: 0.875rem;
  font-weight: 400;
}

.v-list .v-list-item__subtitle.account-name,
.account-name {
  font-size: 0.75rem;
}

.user-avatar {
  color: $app-header-font-color;
  border-radius: 0.15rem;
  background-color: $BCgovBlue3;
  font-size: 1.1875rem;
  font-weight: 400;
}

.log-in-btn {
  color: $BCgovBlue5;
  background-color: $BCgovGold4;
  font-weight: 700;
}

.v-list--dense .v-subheader {
  padding-right: 1rem;
  padding-left: 1rem;
}

.v-subheader {
  color: $gray9 !important;
  font-size: 0.875rem;
  font-weight: 700;
}
</style>
