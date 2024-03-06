<script setup lang="ts">
import { MsgCreateValidator } from "cosmjs-types/cosmos/staking/v1beta1/tx";
import { TxBody, AuthInfo, TxRaw } from "cosmjs-types/cosmos/tx/v1beta1/tx";
import { encodePubkey } from "@cosmjs/proto-signing";
import { Uint32 } from "@cosmjs/math";
import {
  toBech32,
  fromBech32,
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
  description: {
    moniker: "A Node",
    identity: "",
    website: "https://test.iconlake.com/",
    details: "",
    securityContact: "support@iconlake.com",
  },
  commission: {
    maxChangeRate: "0.010000000000000000",
    maxRate: "0.200000000000000000",
    rate: "0.100000000000000000",
  },
  pubkey: "oxJ4lDdSSNHHhQ776UKdMIIhwsaD8YU3v4eLxQMP84s=",
  memo: "",
  minSelfDelegation: "1",
  amount: "1000000LAKE",
  fee: "0.005LAKE",
  delegatorAddress: "",
  validatorAddress: "",
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
  fm.delegatorAddress = signer.address;
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

function toIntRate(rate: string) {
  const r = rate.replace(/^0\./, "").substring(0, 18);
  const fixLen = 18 - r.length;
  return r.replace(/^0*/, "") + "0".repeat(fixLen);
}

function toUlakeAmount(amount: string) {
  return Math.floor(parseFloat(amount) * 1000000).toString();
}

async function sign() {
  const createMsg = MsgCreateValidator.encode({
    description: {
      moniker: fm.description.moniker,
      identity: fm.description.identity,
      website: fm.description.website,
      details: fm.description.details,
      securityContact: fm.description.securityContact,
    },
    commission: {
      maxChangeRate: toIntRate(fm.commission.maxChangeRate), // 0.01
      maxRate: toIntRate(fm.commission.maxRate), // 0.2
      rate: toIntRate(fm.commission.rate), // 0.1
    },
    minSelfDelegation: fm.minSelfDelegation,
    delegatorAddress: fm.delegatorAddress,
    validatorAddress: fm.validatorAddress,
    pubkey: encodePubkey({
      type: "tendermint/PubKeyEd25519",
      value: fm.pubkey,
    }),
    value: {
      amount: toUlakeAmount(fm.amount),
      denom: "ulake",
    },
  }).finish();

  const bodyBytes = TxBody.encode({
    messages: [
      {
        typeUrl: MsgCreateValidator.typeUrl,
        value: createMsg,
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

watch(
  () => fm.delegatorAddress,
  () => {
    fm.validatorAddress = toBech32(
      "iconlakevaloper",
      fromBech32(fm.delegatorAddress).data
    );
  }
);

const signedTX = computed(() => {
  return JSON.stringify(
    {
      body: {
        messages: [
          {
            "@type": "/cosmos.staking.v1beta1.MsgCreateValidator",
            description: {
              moniker: fm.description.moniker,
              identity: fm.description.identity,
              website: fm.description.website,
              security_contact: fm.description.securityContact,
              details: fm.description.details,
            },
            commission: {
              rate: fm.commission.rate,
              max_rate: fm.commission.maxRate,
              max_change_rate: fm.commission.maxChangeRate,
            },
            min_self_delegation: fm.minSelfDelegation,
            delegator_address: fm.delegatorAddress,
            validator_address: fm.validatorAddress,
            pubkey: {
              "@type": "/cosmos.crypto.ed25519.PubKey",
              key: fm.pubkey,
            },
            value: {
              denom: "ulake",
              amount: toUlakeAmount(fm.amount),
            },
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
  while (n++ < 10) {
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
            <div class="help" v-if="kC === 'identity'">
              It can be used to verify identity with systems like Keybase or
              UPort.
            </div>
          </div>
        </div>
      </div>
      <div v-else class="flex">
        <label>{{ k }}</label>
        <div>
          <input type="text" v-model="fm[k]" />
          <div class="help" v-if="k === 'pubkey'">
            Get key by running `./iconlaked tendermint show-validator`.
          </div>
        </div>
      </div>
    </div>
    <div class="btns">
      <button @click="sign">Sign</button>
    </div>
  </div>
  <div v-if="isKeplrReady" class="rs">
    <h4>Signed TX</h4>
    <textarea cols="90" rows="68">{{ signedTX }}</textarea>
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
