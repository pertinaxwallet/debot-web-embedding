<script>
  import { getContext } from "svelte";
  import { Button } from "svelte-chota";
  import SigningBoxClass from "./../../lib/SigningBox";
  import i18n from "./../i18n.js";
  //Context
  const { getEngine } = getContext("app_functions");
  export let element;
  let error;

  let engine = getEngine();

  const useEverWallet = async () => {
    error = "";
    try {
      if (typeof window.hasTonProvider === 'undefined') {
        error = 'Ever wallet extension is not available, please install.';
        return;
      }
     
      const response = await window.ton.request({"method": "requestPermissions", "params": {permissions: ['tonClient', 'accountInteraction']}});

      if (response == null || response.accountInteraction == null) {
        error = 'Insufficient permissions';
        return;
      }

      const publicKey = response.accountInteraction.publicKey;
      const address = response.accountInteraction.address;
      const signer = async (unsigned) => {
        const result = await window.ton.request({"method": "signDataRaw", "params": { publicKey, data: unsigned }});
        console.log(result);
        return {signature: result.signatureHex};
      };
      const SigningBox = new SigningBoxClass(publicKey, signer);
      const SigningBoxHandle = (await engine.client.crypto.register_signing_box(SigningBox)).handle;
      element.setUsed();
      const sg = { signing_box: SigningBoxHandle };
      engine.setSigningBox(sg);
      engine.setAccount({address, publicKey});
      element.resolve(sg);
    } catch (e) {
      error = i18n('tag.selectKey.unknownError');
    }
  }

  const usePertinaxWallet = async () => {
    error = "";
    try {
      if (typeof window.everscale === 'undefined') {
        error = 'Pertinax wallet extension is not available, please install <a href="https://pertinaxwallet.com/installation.html" target="_blank">it here</a>.';
        return;
      }
     
      let response = await window.everscale.request({"method": "wallet_getPermissions", "params": {}});

      if (!response.includes("ever_signMessage")) {
        try {
          response = await window.everscale.request({"method": "wallet_requestPermissions", "params": {"permissions": ["ever_account", "ever_signMessage"]}});
          if (!response.includes("ever_signMessage")) {
            error = 'Insufficient permissions';
          }
          return;
        } catch(e) {
          error = 'Insufficient permissions';
          return;
        }
      }

      let address, publicKey; 
      try {
        response = await window.everscale.request({"method": "ever_account", "params": {}});
        address = response.address;
        publicKey = response.publicKey;
      } catch(e) {
        console.log(e);
        error = 'Insufficient permissions';
      }

      const signer = async (unsigned) => {
        const result = await window.everscale.request({"method": "ever_signMessage", "params": { data: unsigned }});
        console.log(result);
        return {signature: result};
      };
      const SigningBox = new SigningBoxClass(publicKey, signer);
      const SigningBoxHandle = (await engine.client.crypto.register_signing_box(SigningBox)).handle;
      element.setUsed();
      const sg = { signing_box: SigningBoxHandle };
      engine.setSigningBox(sg);
      engine.setAccount({address, publicKey});
      element.resolve(sg);
    } catch (e) {
      error = i18n('tag.selectKey.unknownError');
    }
  }
</script>

<div>{i18n('tag.selectKey.title')}</div>

<div class="is-center">
  <Button size="3" on:click={(() => {useEverWallet()})} disabled={element.isUsed} class="is-rounded" title="Ever wallet" icon="__CDN_URL__/assets/img/ever-wallet.svg"/>
  <Button size="3" on:click={(() => {usePertinaxWallet()})} disabled={element.isUsed} class="is-rounded" title="Pertinax wallet" icon="__CDN_URL__/assets/img/pertinax-wallet.svg"/>
</div>
{#if error}
  <div class="error">{ error }</div>
{/if}