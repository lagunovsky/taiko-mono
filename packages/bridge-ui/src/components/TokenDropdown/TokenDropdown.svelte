<script lang="ts">
  import { deepEqual } from '@wagmi/core';
  import { createEventDispatcher, onDestroy, onMount, tick } from 'svelte';
  import { t } from 'svelte-i18n';
  import type { Address } from 'viem';
  import { zeroAddress } from 'viem';

  import {
    computingBalance,
    destNetwork,
    errorComputingBalance,
    selectedToken,
    tokenBalance,
  } from '$components/Bridge/state';
  import { DesktopOrLarger } from '$components/DesktopOrLarger';
  import { Icon } from '$components/Icon';
  import Erc20 from '$components/Icon/ERC20.svelte';
  import { warningToast } from '$components/NotificationToast';
  import { OnAccount } from '$components/OnAccount';
  import { tokenService } from '$libs/storage/services';
  import { ETHToken, fetchBalance as getTokenBalance, type NFT, type Token, TokenType } from '$libs/token';
  import { getTokenAddresses } from '$libs/token/getTokenAddresses';
  import { getLogger } from '$libs/util/logger';
  import { truncateString } from '$libs/util/truncateString';
  import { type Account, account } from '$stores/account';
  import { connectedSourceChain } from '$stores/network';

  import AddCustomErc20 from './AddCustomERC20.svelte';
  import DialogView from './DialogView.svelte';
  import DropdownView from './DropdownView.svelte';
  import { symbolToIconMap } from './symbolToIconMap';
  import { TabTypes } from './types';

  const log = getLogger('TokenDropdown');

  const dispatch = createEventDispatcher();

  export let tokens: Token[] = [];
  export let value: Maybe<Token | NFT> = null;
  export let onlyMintable: boolean = false;
  export let disabled = false;
  export let combined = false;

  let customTokenModalOpen = false;

  let id = `menu-${crypto.randomUUID()}`;
  $: menuOpen = false;

  let activeTab: TabTypes = TabTypes.TOKEN;

  let customTokens = tokenService.getTokens($account?.address as Address);

  // This will control which view to render depending on the screensize.
  // Since markup will differ, and there is logic running when interacting
  // with this component, it makes more sense to not render the view that's
  // not being used, doing this with JS instead of CSS media queries
  let isDesktopOrLarger: boolean;

  const closeMenu = () => {
    menuOpen = false;
  };

  const openMenu = () => {
    menuOpen = true;
  };

  const selectToken = async (token: Token) => {
    dispatch('tokenSelected', { token });
    const srcChain = $connectedSourceChain;
    const destChain = $destNetwork;
    $computingBalance = true;
    closeMenu();
    log('selected token', token);
    if (token === value) {
      log('same token, nothing to do');
      // same token, nothing to do
      $computingBalance = false;
      return;
    }

    // In order to select a token, we only need the source chain to be selected,
    if (!srcChain) {
      warningToast({ title: $t('messages.network.required') });
      $computingBalance = false;
      return;
    }
    if (!destChain || !destChain.id) {
      warningToast({ title: $t('messages.network.required_dest') });
      $computingBalance = false;
      return;
    }
    try {
      const tokenInfo = await getTokenAddresses({ token, srcChainId: srcChain.id, destChainId: destChain.id });
      if (!tokenInfo) {
        $computingBalance = false;
      } else {
        if (tokenInfo.bridged?.chainId && tokenInfo.bridged?.address && tokenInfo.bridged?.address !== zeroAddress) {
          token.addresses[tokenInfo.bridged.chainId] = tokenInfo.bridged.address;
          tokenService.updateToken(token, $account?.address as Address);
        }
      }
    } catch (error) {
      $computingBalance = false;
      console.error(error);
    }
    value = token;
    await updateBalance();
    $computingBalance = false;
  };

  const handleCustomTokenModal = () => {
    closeMenu();
    customTokenModalOpen = true;
  };

  const handleTokenRemoved = (event: { detail: { token: Token } }) => {
    const token = event.detail.token;

    // if the selected token is the one that was removed by the user, remove it
    if (deepEqual(token, value)) {
      value = ETHToken;
    }
    const address = $account.address;
    tokenService.removeToken(token, address as Address);
    customTokens = tokenService.getTokens(address as Address);
  };

  async function updateBalance() {
    const userAddress = $account?.address;
    const srcChainId = $connectedSourceChain?.id;
    const destChainId = $destNetwork?.id;
    const token = value;
    if (!token || !srcChainId || !userAddress) return;
    $computingBalance = true;
    $errorComputingBalance = false;

    try {
      if (token.type === TokenType.ERC20) {
        $tokenBalance = await getTokenBalance({
          token,
          srcChainId,
          destChainId,
          userAddress,
        });
      } else if (token.type === TokenType.ETH) {
        $tokenBalance = await getTokenBalance({
          token: ETHToken,
          srcChainId,
          destChainId,
          userAddress,
        });
      } else {
        $tokenBalance = await getTokenBalance({
          token,
          srcChainId,
          destChainId,
          userAddress,
        });
      }
    } catch (err) {
      log('Error updating balance: ', err);
      //most likely we have a custom token that is not bridged yet
      $errorComputingBalance = true;
      // clearAmount();
    }
    $computingBalance = false;
  }

  const onAccountChange = (newAccount: Account, prevAccount?: Account) => {
    if (newAccount?.chainId === prevAccount?.chainId || !newAccount || !prevAccount) reset();
  };

  const reset = async () => {
    const srcChain = $connectedSourceChain;
    const destChain = $destNetwork;
    const user = $account?.address;
    $selectedToken = ETHToken;
    tick();
    if (!srcChain || !destChain || !user) return;
    $computingBalance = true;
    value = tokens[0];
    $selectedToken = value;
    $computingBalance = false;
  };

  $: textClass = disabled ? 'text-secondary-content' : 'font-bold ';

  onDestroy(() => closeMenu());

  onMount(async () => {
    reset();
  });
</script>

<!-- svelte-ignore missing-declaration -->
<DesktopOrLarger bind:is={isDesktopOrLarger} />

<div class="relative h-full {$$props.class}">
  <button
    {disabled}
    aria-haspopup="listbox"
    aria-controls={id}
    aria-expanded={menuOpen}
    class="f-between-center w-full h-full px-[20px] py-[14px] input-box bg-neutral-background border-0 shadow-none outline-none
    {combined ? '!rounded-l-[0px] !rounded-r-[10px]' : '!rounded-[10px]'}"
    on:click|stopPropagation={openMenu}>
    <div class="space-x-2">
      {#if !value || disabled}
        <span class="title-subsection-bold text-base text-secondary-content">{$t('token_dropdown.label')}</span>
      {:else if value}
        <div class="flex f-space-between space-x-2 items-center text-secondary-content">
          <!-- Only match icons to configured tokens -->
          {#if symbolToIconMap[value.symbol] && !value.imported}
            <i role="img" aria-label={value.name}>
              <svelte:component this={symbolToIconMap[value.symbol]} size={20} />
            </i>
          {:else if value.logoURI}
            <img src={value.logoURI} alt={value.name} class="w-[20px] h-[20px] rounded-[50%]" />
          {:else}
            <i role="img" aria-label={value.symbol}>
              <svelte:component this={Erc20} size={20} />
            </i>
          {/if}
          <span class={textClass}>{truncateString(value.symbol, 6)}</span>
        </div>
      {/if}
    </div>
    {#if !disabled}
      <Icon type="chevron-down" size={10} />
    {/if}
  </button>
  {#if isDesktopOrLarger}
    <DropdownView
      {id}
      bind:menuOpen
      {onlyMintable}
      {tokens}
      bind:customTokens
      {value}
      {selectToken}
      {closeMenu}
      {activeTab}
      on:tokenRemoved={handleTokenRemoved}
      on:openCustomTokenModal={handleCustomTokenModal} />
  {:else}
    <DialogView
      {id}
      bind:menuOpen
      {onlyMintable}
      {tokens}
      bind:customTokens
      {value}
      {selectToken}
      {closeMenu}
      {activeTab}
      on:tokenRemoved={handleTokenRemoved}
      on:openCustomTokenModal={handleCustomTokenModal} />
  {/if}
</div>

<div data-modal-uuid={id} />

<OnAccount change={onAccountChange} />

<AddCustomErc20 bind:modalOpen={customTokenModalOpen} bind:customTokens on:tokenRemoved />
