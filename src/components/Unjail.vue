<script setup lang="ts">
import { MsgUnjail } from "cosmjs-types/cosmos/slashing/v1beta1/tx";
import { TxBody, AuthInfo, TxRaw } from "cosmjs-types/cosmos/tx/v1beta1/tx";
import { encodePubkey } from "@cosmjs/proto-signing";
import { Uint32 } from "@cosmjs/math";
import {
  toBase64,
  fromBase64,
  toHex,
} from "@cosmjs/encoding";
import { Window, Keplr, AccountData, BroadcastMode } from "@keplr-wallet/types";
import {
  computed,
  onBeforeUnmount,
  onMounted,
  reactive,
  ref,
  watch,
} from "vue";
import Long from "long";

let keplr: Keplr;
let signer: AccountData;

const isKeplrReady = ref(false);
const fm = reactive({
  validatorAddr: '',
  fee: "0.005LAKE",
  memo: "",
  accountNumber: "12",
  sequence: "0",
  chainId: "iconlake-testnet-1",
});
const signature = ref("");
let txBytes = ref(new Uint8Array());

async function init() {
  isKeplrReady.value = !!(window as Window).keplr;
  if (!isKeplrReady.value) {
    return;
  }
  keplr = (window as Window).keplr!;
  await keplr?.enable(fm.chainId);
  signer = (await keplr.getOfflineSigner(fm.chainId).getAccounts())[0];
  fm.validatorAddr = signer.address;
  const res = await fetch(
    `https://lcd.testnet.iconlake.com/cosmos/auth/v1beta1/accounts/${signer.address}`
  ).then((e) => e.json());
  if (res.account) {
    fm.accountNumber = res.account.account_number;
    fm.sequence = res.account.sequence;
  }
}

function onKeplrChange() {
  init();
}

function toUlakeAmount(amount: string) {
  return Math.floor(parseFloat(amount) * 1000000).toString();
}

async function sign() {
  const msg = MsgUnjail.encode({
    validatorAddr: fm.validatorAddr,
  }).finish();

  const bodyBytes = TxBody.encode({
    messages: [
      {
        typeUrl: MsgUnjail.typeUrl,
        value: msg,
      },
    ],
    memo: fm.memo,
    timeoutHeight: new Uint32(0).toBigInt(),
    extensionOptions: [],
    nonCriticalExtensionOptions: [],
  }).finish();

  const authInfoBytes = AuthInfo.encode({
    signerInfos: [
      {
        publicKey: encodePubkey({
          type: "tendermint/PubKeySecp256k1",
          value: toBase64(signer.pubkey),
        }),
        modeInfo: {
          single: {
            mode: 1,
          },
        },
        sequence: new Uint32(+fm.sequence).toBigInt(),
      },
    ],
    fee: {
      amount:
        toUlakeAmount(fm.fee) === "0"
          ? []
          : [
              {
                denom: "ulake",
                amount: toUlakeAmount(fm.fee),
              },
            ],
      gasLimit: new Uint32(200000).toBigInt(),
      payer: "",
      granter: "",
    },
  }).finish();

  const res = await keplr.signDirect(
    fm.chainId,
    signer.address,
    {
      bodyBytes,
      authInfoBytes,
      chainId: fm.chainId,
      accountNumber: new Long(+fm.accountNumber),
    },
    {
      preferNoSetFee: true,
      preferNoSetMemo: true,
      disableBalanceCheck: true,
    }
  );

  signature.value = res.signature.signature;

  txBytes.value = TxRaw.encode({
    bodyBytes: bodyBytes,
    authInfoBytes: authInfoBytes,
    signatures: [fromBase64(signature.value)],
  }).finish();

  broadStatus.value = 0;
}

function isObject(v: any) {
  return v.toString() === "[object Object]";
}

onMounted(() => {
  init();
  window.addEventListener("keplr_keystorechange", onKeplrChange);
});

onBeforeUnmount(() => {
  window.removeEventListener("keplr_keystorechange", onKeplrChange);
});

watch(() => fm.chainId, init);

const signedTX = computed(() => {
  return JSON.stringify(
    {
      body: {
        messages: [
          {
            "@type": "/cosmos.slashing.v1beta1.MsgUnjail",
            validator_addr: fm.validatorAddr,
          },
        ],
        memo: fm.memo,
        timeout_height: "0",
        extension_options: [],
        non_critical_extension_options: [],
      },
      auth_info: {
        signer_infos: [
          {
            public_key: {
              "@type": "/cosmos.crypto.secp256k1.PubKey",
              key: signer ? toBase64(signer.pubkey) : "",
            },
            mode_info: {
              single: {
                mode: "SIGN_MODE_DIRECT",
              },
            },
            sequence: "0",
          },
        ],
        fee: {
          amount:
            toUlakeAmount(fm.fee) === "0"
              ? []
              : [
                  {
                    denom: "ulake",
                    amount: toUlakeAmount(fm.fee),
                  },
                ],
          gas_limit: "200000",
          payer: "",
          granter: "",
        },
        tip: null,
      },
      signatures: [signature.value],
    },
    undefined,
    2
  );
});

const broadStatus = ref(0);
const txHash = ref("");
const txResult = ref("");
const txResultMsg = ref("");

async function broadcast() {
  broadStatus.value = 1;
  const txHashBytes = await keplr
    .sendTx(fm.chainId, txBytes.value, "async" as BroadcastMode)
    .catch((e) => {
      console.error(e);
      broadStatus.value = 4;
      txResult.value = e.message;
    });
  if (!txHashBytes) {
    return;
  }
  txHash.value = toHex(txHashBytes);
  broadStatus.value = 2;
  let txRes;
  let n = 0;
  while (n++ < 30) {
    txRes = await fetch(
      `https://lcd.testnet.iconlake.com/cosmos/tx/v1beta1/txs/${txHash.value}`
    )
      .then((e) => e.json())
      .catch((e) => {
        console.error(e);
        broadStatus.value = 4;
        txResultMsg.value = e.message;
        txResult.value = e.message;
      });
    if (txRes.tx) {
      break;
    }
    await new Promise((resolve) => setTimeout(resolve, 1000));
  }
  if (!txRes) {
    return;
  }
  broadStatus.value = 3;
  txResultMsg.value = txRes.tx_response?.raw_log || txRes.message;
  txResult.value = JSON.stringify(txRes, undefined, 2);
}
</script>

<template>
  <div v-if="!isKeplrReady">Keplr is not installed.</div>
  <div class="fm" v-else>
    <div v-for="(v, k) in fm">
      <div v-if="isObject(v)" class="group">
        <h5>{{ k }}</h5>
        <div class="flex" v-for="(vC, kC) in v">
          <label>{{ kC }}</label>
          <div>
            <input type="text" v-model="fm[k][kC]" />
          </div>
        </div>
      </div>
      <div v-else class="flex">
        <label>{{ k }}</label>
        <div>
          <input type="text" v-model="fm[k]" />
        </div>
      </div>
    </div>
    <div class="btns">
      <button @click="sign">Sign</button>
    </div>
  </div>
  <div v-if="isKeplrReady" class="rs">
    <h4>Signed TX</h4>
    <textarea cols="90" rows="48">{{ signedTX }}</textarea>
    <div class="btns" v-if="broadStatus === 0 || broadStatus === 4">
      <button @click="broadcast">Broadcast</button>
    </div>
    <div class="msg" v-else-if="broadStatus === 1">Broading</div>
    <div class="msg" v-if="broadStatus === 2">
      <p>TxHash: {{ txHash }}</p>
      <p>Waiting for confirmation...</p>
    </div>
    <div class="msg" v-if="broadStatus === 3">
      <p><b>TxResult:</b> {{ txResultMsg }}</p>
      <pre>{{ txResult }}</pre>
    </div>
    <div class="msg" v-if="broadStatus === 4">Error: {{ txResult }}</div>
  </div>
</template>

<style lang="scss" scoped>
body {
  margin: 0;
}

.flex {
  display: flex;
  margin-top: 16px;
}

.group {
  margin-bottom: 50px;
}

.btns {
  margin-top: 50px;
}

.fm {
  label,
  h5 {
    width: 220px;
    text-align: right;
    margin-right: 12px;
  }

  input {
    width: 400px;
  }
}

.help {
  text-align: left;
  font-size: 12px;
}

.rs {
  padding-bottom: 50px;
  h4 {
    text-align: left;
  }
}

.msg {
  margin-top: 16px;
  width: 745px;
  overflow: auto;
  pre {
    text-align: left;
  }
}
</style>
