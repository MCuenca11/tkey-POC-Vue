<template>
  <div id="app">
    <p class="font-italic">This is a POC for integrating tkey with Visa Guide. Please open console for more detailed info.</p>
    <div>
      <span :style="{ marginRight: '20px' }">To Begin Select Social Provider (Or Select Mocked Login) and Initialize New tkey to Login. (After tkey Has Been Created, You Can Login Directly With Login Button):</span>
    </div>
    <select v-model="selectedVerifier">
      <option :key="login" v-for="login in Object.keys(verifierMap)" :value="login">{{ verifierMap[login].name }}</option>
    </select>
    <div :style="{ marginTop: '20px' }" v-if="selectedVerifier === 'passwordless'">
      <input type="email" v-model="loginHint" placeholder="Enter your email" />
    </div>

    <div :style="{ marginTop: '20px' }">
      <input type="checkbox" id="mocked" v-model="mocked" />
      <label for="mocked">Mocked Login</label>
    </div>

    <div :style="{ marginTop: '20px' }">
      <h4>tKey Init and Login</h4>
      <div :style="{ margin: '20px' }">
        <input v-model="answer1" placeholder="Enter Visa Guide ID" />
      </div>
      <div :style="{ margin: '20px' }">
        <input v-model="answer2" placeholder="Enter Provider ID" />
      </div>
        <button @click="generateNewShareWithSecurityQuestions">Initialize New tkey Using 3 Shares</button>
        <button @click="reconstructWithIDandDevice">Input Your Visa Guide ID to Login with Guide + Device Shares</button>
        <button @click="generateNewShare">Generate New Share</button>
      <br />
      <h4>Lost Share Recovery Using Social Provider</h4>
        <!-- <button @click="_initializeNewKey">Create New tkey Using Social Provider/Device</button> -->
        <button @click="recoverDeviceShare">Recover tkey using Provider and Visa Guide ID</button>
        <button @click="reconstructWithProviderAndDevice">Recover tkey using Provider and Device</button>
      <br />
      <h4>Get tkey Info</h4>
        <!-- <button @click="reconstructKey">Reconstuct tkey (Don't Need Anymore?)</button> -->
        <button @click="getKeyDetails">Get tkey Details</button>
        <button @click="getSDKObject">Get tkey Object</button>
      <br />
      <h4>Share Transer (Not sure if needed)</h4>
        <button @click="checkShareRequests">Check Share Requests</button>
        <button @click="requestShare">Request Share</button>
        <button @click="approveShareRequest">Approve Request</button>
        <button @click="resetShareRequests">Reset Share Request Store</button>
    </div>
      <a target="_blank" href="https://github.com/MCuenca11/tkey-for-POC" rel="noreferrer">
            Click Here to Access the Source Code
      </a>
    <div id="console">
      <p></p>
    </div>
  </div>
</template>

<script>
import ThresholdKey from "@tkey/default";
import TorusServiceProvider from "@tkey/service-provider-torus";
import TorusStorageLayer from "@tkey/storage-layer-torus";
import ServiceProviderBase from "@tkey/service-provider-base";
import WebStorageModule from "@tkey/web-storage";
import SecurityQuestionsModule from "@tkey/security-questions";
import ShareTransferModule from "@tkey/share-transfer";

const GOOGLE = "google";
const FACEBOOK = "facebook";
const REDDIT = "reddit";
const DISCORD = "discord";
const TWITCH = "twitch";
const GITHUB = "github";
const APPLE = "apple";
const LINKEDIN = "linkedin";
const TWITTER = "twitter";
const WEIBO = "weibo";
const LINE = "line";
const EMAIL_PASSWORD = "email_password";
const PASSWORDLESS = "passwordless";
const HOSTED_EMAIL_PASSWORDLESS = "hosted_email_passwordless";
const HOSTED_SMS_PASSWORDLESS = "hosted_sms_passwordless";

const AUTH_DOMAIN = "https://torus-test.auth0.com";

export default {
  name: "App",
  data() {
    return {
      torusdirectsdk: undefined,
      selectedVerifier: "google",
      loginHint: "",
      mocked: true,
      answer1: "",
      answer2: "",
      verifierMap: {
        [GOOGLE]: {
          name: "Google",
          typeOfLogin: "google",
          clientId: "221898609709-obfn3p63741l5333093430j3qeiinaa8.apps.googleusercontent.com",
          verifier: "google-lrc"
        },
        [FACEBOOK]: { name: "Facebook", typeOfLogin: "facebook", clientId: "617201755556395", verifier: "facebook-lrc" },
        [REDDIT]: { name: "Reddit", typeOfLogin: "reddit", clientId: "YNsv1YtA_o66fA", verifier: "torus-reddit-test" },
        [TWITCH]: { name: "Twitch", typeOfLogin: "twitch", clientId: "f5and8beke76mzutmics0zu4gw10dj", verifier: "twitch-lrc" },
        [DISCORD]: { name: "Discord", typeOfLogin: "discord", clientId: "682533837464666198", verifier: "discord-lrc" },
        [EMAIL_PASSWORD]: {
          name: "Email Password",
          typeOfLogin: "email_password",
          clientId: "sqKRBVSdwa4WLkaq419U7Bamlh5vK1H7",
          verifier: "torus-auth0-email-password"
        },
        [PASSWORDLESS]: {
          name: "Passwordless",
          typeOfLogin: "passwordless",
          clientId: "P7PJuBCXIHP41lcyty0NEb7Lgf7Zme8Q",
          verifier: "torus-auth0-passwordless"
        },
        [APPLE]: { name: "Apple", typeOfLogin: "apple", clientId: "m1Q0gvDfOyZsJCZ3cucSQEe9XMvl9d9L", verifier: "torus-auth0-apple-lrc" },
        [GITHUB]: { name: "Github", typeOfLogin: "github", clientId: "PC2a4tfNRvXbT48t89J5am0oFM21Nxff", verifier: "torus-auth0-github-lrc" },
        [LINKEDIN]: { name: "Linkedin", typeOfLogin: "linkedin", clientId: "59YxSgx79Vl3Wi7tQUBqQTRTxWroTuoc", verifier: "torus-auth0-linkedin-lrc" },
        [TWITTER]: { name: "Twitter", typeOfLogin: "twitter", clientId: "A7H8kkcmyFRlusJQ9dZiqBLraG2yWIsO", verifier: "torus-auth0-twitter-lrc" },
        [WEIBO]: { name: "Weibo", typeOfLogin: "weibo", clientId: "dhFGlWQMoACOI5oS5A1jFglp772OAWr1", verifier: "torus-auth0-weibo-lrc" },
        [LINE]: { name: "Line", typeOfLogin: "line", clientId: "WN8bOmXKNRH1Gs8k475glfBP5gDZr9H1", verifier: "torus-auth0-line-lrc" },
        [HOSTED_EMAIL_PASSWORDLESS]: {
          name: "Hosted Email Passwordless",
          typeOfLogin: "jwt",
          clientId: "P7PJuBCXIHP41lcyty0NEb7Lgf7Zme8Q",
          verifier: "torus-auth0-passwordless"
        },
        [HOSTED_SMS_PASSWORDLESS]: {
          name: "Hosted SMS Passwordless",
          typeOfLogin: "jwt",
          clientId: "nSYBFalV2b1MSg5b2raWqHl63tfH3KQa",
          verifier: "torus-auth0-sms-passwordless"
        }
      }
    };
  },
  computed: {
    loginToConnectionMap() {
      return {
        [EMAIL_PASSWORD]: { domain: AUTH_DOMAIN },
        [PASSWORDLESS]: { domain: AUTH_DOMAIN, login_hint: this.loginHint },
        [HOSTED_EMAIL_PASSWORDLESS]: { domain: AUTH_DOMAIN, verifierIdField: "name", connection: "", isVerifierIdCaseSensitive: false },
        [HOSTED_SMS_PASSWORDLESS]: { domain: AUTH_DOMAIN, verifierIdField: "name", connection: "" },
        [APPLE]: { domain: AUTH_DOMAIN },
        [GITHUB]: { domain: AUTH_DOMAIN },
        [LINKEDIN]: { domain: AUTH_DOMAIN },
        [TWITTER]: { domain: AUTH_DOMAIN },
        [WEIBO]: { domain: AUTH_DOMAIN },
        [LINE]: { domain: AUTH_DOMAIN }
      };
    }
  },

  methods: {
    getKeyDetails() {
      console.log(this.tbsdk.getKeyDetails());
      this.console("Key Details: " + JSON.stringify(this.tbsdk.getKeyDetails()));
      return this.tbsdk.getKeyDetails();
    },
    getLatestPolynomialDetails() {
      const metadata = this.tbSDK.getMetadata();
      let latestPolynomial = metadata.getLatestPublicPolynomial();
      let latestPolynomialId = latestPolynomial.getPolynomialID();
      let indexes = metadata.getShareIndexesForPolynomial(latestPolynomialId);
      console.log(latestPolynomial, latestPolynomialId, indexes);
    },
    getSDKObject() {
      this.console("Key Object: " + JSON.stringify(this.tbsdk));
      console.log(this.tbsdk);
    },
    passwordValidation(v) {
      return v.length >= 5;
      // return /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^\dA-Za-z]).\S{10,}$/.test(v)
    },
    async initializeAndReconstruct() {
      try {
        const initializedDetails = await this.tbsdk.initialize();
        console.log(initializedDetails);

        let shareDesc = Object.assign({}, initializedDetails.shareDescriptions);
        Object.keys(shareDesc).map(el => {
          shareDesc[el] = shareDesc[el].map(jl => {
            return JSON.parse(jl);
          });
        });
        console.log(shareDesc);

        // Check different types of shares from metadata. This helps in making UI decisions (About what kind of shares to ask from users)
        // Sort the share descriptions with priority order
        let priorityOrder = ["securityQuestions", "webStorage"];

        let tempSD = Object.values(shareDesc)
          .flatMap(x => x)
          .sort((a, b) => {
            return priorityOrder.indexOf(a.module) - priorityOrder.indexOf(b.module);
          });

        if (tempSD.length === 0 && requiredShares > 0) {
          throw new Error("No share descriptions available. New key assign might be required or contact support");
        }
        let requiredShares = initializedDetails.requiredShares;
        while (requiredShares > 0 && tempSD.length > 0) {
          let currentPriority = tempSD.shift();
          if (currentPriority.module === "webStorage") {
            try {
              await this.tbsdk.modules.webStorage.inputShareFromWebStorage();
              console.log("Getting Device Storage Share...");
              requiredShares--;
            } catch (err) {
              this.console("Couldn't Find The Device Share");
              console.log("Couldn't Find The Device Share");
            }
          } else if (currentPriority.module === "securityQuestions") {
            // default to password for now
            console.log("Logging in with Visa Guide ID...");
            // await this.tbsdk.modules.securityQuestions.inputShareFromSecurityQuestions(this.answer, "What's your Visa Guide ID?");
            // throw "Logging in with Visa Guide...";
          } 

          if (tempSD.length === 0 && requiredShares > 0) {
            throw "URGENT: Need to Reassign Lost Key!";
          }
        }

        console.log(this.tbsdk);

        // const key = await this.tbsdk.reconstructKey();
        // await this.tbsdk._initializeNewKey(undefined, true)
        console.log("Logged In Successfully!");
        console.log(initializedDetails);
        // console.log(key.privKey.toString("hex"));
        // this.console(key);

        // this.console(initializedDetails);
      } catch (error) {
        console.error(error, "caught");
      }
    },
    async triggerLogin() {
      try {
        await this.initTkey();

        // console.log(this.tbsdk, this.tbsdk.serviceProvider, this.tbsdk.serviceProvider.__proto__ )
        // If the service providers aren't set up then return
        if (!this.tbsdk.serviceProvider) return;
        // if it's not mocked then use the social providers
        if (!this.mocked) {
          const jwtParams = this.loginToConnectionMap[this.selectedVerifier] || {};
          const { typeOfLogin, clientId, verifier } = this.verifierMap[this.selectedVerifier];
          // await this.tbsdk.serviceProvider.triggerLogin({
          //   typeOfLogin,
          //   verifier,
          //   clientId,
          //   jwtParams
          // });

          await this.tbsdk.serviceProvider.triggerHybridAggregateLogin({
            singleLogin: {
              typeOfLogin,
              verifier,
              clientId,
              jwtParams
            },
            aggregateLoginParams: {
              aggregateVerifierType: "single_id_verifier",
              verifierIdentifier: "tkey-google",
              subVerifierDetailsArray: [
                {
                  clientId: "221898609709-obfn3p63741l5333093430j3qeiinaa8.apps.googleusercontent.com",
                  typeOfLogin: "google",
                  verifier: "torus"
                }
              ]
            }
          });
        }

        await this.initializeAndReconstruct();
        this.console("Successfully Logged In!");
      } catch (error) {
        this.console("Login Was Not Successful");
        console.error(error, "triggerLogin() Error");
      }
    },
    async generateNewShareWithSecurityQuestions() {
      try {
        if (!this.passwordValidation(this.answer1)) {
          this.console("Visa Guide ID Can't Be Less Than 5 Characters");
          throw "Visa Guide ID Can't Be Less Than 5 Characters";
        }
        if (!this.passwordValidation(this.answer2)) {
          this.console("Visa Guide ID Can't Be Less Than 5 Characters");
          throw "Visa Guide ID Can't Be Less Than 5 Characters";
        }
        await this.initTkey();
        const res = await this.tbsdk._initializeNewKey();

        // This calls the function in the Security Questions Module
        await this.tbsdk.modules.visaGuide.generateNewShareWithSecurityQuestions(this.answer1, "What's Your Visa Guide ID?");
        console.log("Visa Share Successful");
        await this.tbsdk.modules.socialProvider.generateNewShareWithSecurityQuestions(this.answer2, "What's Your Social Provider ID?");
        console.log("Social Provider Share Successful");
        console.log("New tkey Info:", res);
        this.console("Successfully Initialized Visa Guide ID Share + Device Share + Provider Share! New tkey Info: " + JSON.stringify(res));
        console.log(this.tbsdk.getKeyDetails());
      } catch (error) {
        this.console("Failed to Initialize Share With Visa Guide ID");
        console.error(error, "genNewShareWSecQs() Error");
      }
    },
    async checkShareRequests() {
      try {
        const result = await this.tbsdk.modules.shareTransfer.getShareTransferStore();
        const requests = await this.tbsdk.modules.shareTransfer.lookForRequests();
        console.log(result, requests);
      } catch (err) {
        console.log(err);
      }
    },
    async resetShareRequests() {
      try {
        await this.tbsdk.modules.shareTransfer.resetShareTransferStore();
        this.console("Reset share transfer store successfully");
      } catch (err) {
        console.log(err);
      }
    },
    async approveShareRequest() {
      try {
        const result = await this.tbsdk.modules.shareTransfer.getShareTransferStore();
        const requests = await this.tbsdk.modules.shareTransfer.lookForRequests();
        let shareToShare;
        try {
          shareToShare = await this.tbsdk.modules.webStorage.getDeviceShare();
        } catch (err) {
          console.error("No on device share found. Generating a new share");
          const newShare = await this.tbsdk.generateNewShare();
          shareToShare = newShare.newShareStores[newShare.newShareIndex.toString("hex")];
        }
        console.log(result, requests, this.tbsdk);

        await this.tbsdk.modules.shareTransfer.approveRequest(requests[0], shareToShare);
        this.console("approved");
      } catch (err) {
        console.error(err);
      }
    },
    async requestShare() {
      try {
        const result = await this.tbsdk.modules.shareTransfer.requestNewShare();
        console.log(result);
      } catch (err) {
        console.error(err);
      }
    },
    async reconstructKey() {
      try {
        if (this.tbsdk.requiredShares > 1) {
          this.console("Not Enough Shares to Reconstruct the Key");
          throw "Not Enough Shares to Reconstruct the Key";
        } else {
          let key = await this.tbsdk.reconstructKey();
          this.console("Logged In Successfully! Here's Your Private Key: " + JSON.stringify(key));
          console.log(JSON.stringify(key), this.tbsdk.getKeyDetails());
        }
      } catch (error) {
        console.error(error, "reconstructKey() Error");
      }
    },
    async reconstructWithIDandDevice() {
       try {
        const initializedDetails = await this.tbsdk.initialize();
        console.log(initializedDetails);

        let shareDesc = Object.assign({}, initializedDetails.shareDescriptions);
        Object.keys(shareDesc).map(el => {
          shareDesc[el] = shareDesc[el].map(jl => {
            return JSON.parse(jl);
          });
        });
        console.log(shareDesc);

        // Check different types of shares from metadata. This helps in making UI decisions (About what kind of shares to ask from users)
        // Sort the share descriptions with priority order
        let priorityOrder = ["visaGuide", "webStorage"];

        let tempSD = Object.values(shareDesc)
          .flatMap(x => x)
          .sort((a, b) => {
            return priorityOrder.indexOf(a.module) - priorityOrder.indexOf(b.module);
          });

        if (tempSD.length === 0 && requiredShares > 0) {
          throw new Error("No share descriptions available. New key assign might be required or contact support");
        }
        let requiredShares = initializedDetails.requiredShares;
        let actualRequiredShares = 2;
        while (requiredShares > 0 && tempSD.length > 0) {
          let currentPriority = tempSD.shift();
          if (currentPriority.module === "webStorage") {
            try {
              await this.tbsdk.modules.webStorage.inputShareFromWebStorage();
              console.log("Getting Device Storage Share...");
              requiredShares--;
              actualRequiredShares--;
            } catch (err) {
              this.console("Couldn't Find The Device Share");
              console.log("Couldn't Find The Device Share");
              console.log(initializedDetails);
            }
          } else if (currentPriority.module === "visaGuide") {
            await this.tbsdk.modules.visaGuide.inputShareFromSecurityQuestions(this.answer1, "What's your Visa Guide ID?");
            actualRequiredShares--;
            console.log("Logging in with Visa Guide ID...");
          } 

          if (tempSD.length === 0 && requiredShares > 0) {
            throw "URGENT: Need to Reassign Lost Key!";
          }
        }

        console.log(this.tbsdk);
        console.log(initializedDetails);
        if (actualRequiredShares > 0) {
          this.console('Not Enough Shares (Wrong Visa Guide ID or No Device Share Found');
          throw "Not Enough Shares";
        } else {
          let key = await this.tbsdk.reconstructKey();
          this.console("Logged In Successfully! Here's Your Private Key: " + JSON.stringify(key) + "       Here's The Information For The Shares:    " + JSON.stringify(shareDesc));
          console.log(JSON.stringify(key), this.tbsdk.getKeyDetails());
        }
      } catch (error) {
        console.error(error, "caught");
      }
    },
    async reconstructWithProviderAndDevice() {
       try {
        const initializedDetails = await this.tbsdk.initialize();
        console.log(initializedDetails);

        let shareDesc = Object.assign({}, initializedDetails.shareDescriptions);
        Object.keys(shareDesc).map(el => {
          shareDesc[el] = shareDesc[el].map(jl => {
            return JSON.parse(jl);
          });
        });
        console.log(shareDesc);

        // Check different types of shares from metadata. This helps in making UI decisions (About what kind of shares to ask from users)
        // Sort the share descriptions with priority order
        let priorityOrder = ["socialProvider", "webStorage"];

        let tempSD = Object.values(shareDesc)
          .flatMap(x => x)
          .sort((a, b) => {
            return priorityOrder.indexOf(a.module) - priorityOrder.indexOf(b.module);
          });

        if (tempSD.length === 0 && requiredShares > 0) {
          throw new Error("No share descriptions available. New key assign might be required or contact support");
        }
        let requiredShares = initializedDetails.requiredShares;
        let actualRequiredShares = 2;
        while (requiredShares > 0 && tempSD.length > 0) {
          let currentPriority = tempSD.shift();
          if (currentPriority.module === "webStorage") {
            try {
              await this.tbsdk.modules.webStorage.inputShareFromWebStorage();
              console.log("Getting Device Storage Share...");
              requiredShares--;
              actualRequiredShares--;
            } catch (err) {
              this.console("Couldn't Find The Device Share");
              console.log("Couldn't Find The Device Share");
              console.log(initializedDetails);
            }
          } else if (currentPriority.module === "socialProvider") {
            await this.tbsdk.modules.socialProvider.inputShareFromSecurityQuestions(this.answer2, "What's your Social Provider ID?");
            actualRequiredShares--;
            console.log("Logging in with Social Provider ID...");
          } 

          if (tempSD.length === 0 && requiredShares > 0) {
            throw "URGENT: Need to Reassign Lost Key!";
          }
        }

        console.log(this.tbsdk);
        console.log(initializedDetails);
        if (actualRequiredShares > 0) {
          this.console('Not Enough Shares (Wrong Visa Guide ID or No Device Share Found');
          throw "Not Enough Shares";
        } else {
          let key = await this.tbsdk.reconstructKey();
          this.console("Logged In Successfully! Here's Your Private Key: " + JSON.stringify(key) + " Here's The Information For The Shares: " + JSON.stringify(shareDesc));
          console.log(JSON.stringify(key), this.tbsdk.getKeyDetails());
        }
      } catch (error) {
        console.error(error, "caught");
      }
    },
    async recoverDeviceShare() {
       try {
        const initializedDetails = await this.tbsdk.initialize();
        console.log(initializedDetails);

        let shareDesc = Object.assign({}, initializedDetails.shareDescriptions);
        Object.keys(shareDesc).map(el => {
          shareDesc[el] = shareDesc[el].map(jl => {
            return JSON.parse(jl);
          });
        });
        console.log(shareDesc);

        // Check different types of shares from metadata. This helps in making UI decisions (About what kind of shares to ask from users)
        // Sort the share descriptions with priority order
        let priorityOrder = ["visaGuide", "socialProvider"];

        let tempSD = Object.values(shareDesc)
          .flatMap(x => x)
          .sort((a, b) => {
            return priorityOrder.indexOf(a.module) - priorityOrder.indexOf(b.module);
          });

        if (tempSD.length === 0 && requiredShares > 0) {
          throw new Error("No share descriptions available. New key assign might be required or contact support");
        }
        let requiredShares = initializedDetails.requiredShares;
        let actualRequiredShares = 2;
        while (requiredShares > 0 && tempSD.length > 0) {
          let currentPriority = tempSD.shift();
          if (currentPriority.module === "visaGuide") {
            await this.tbsdk.modules.visaGuide.inputShareFromSecurityQuestions(this.answer1, "What's your Visa Guide ID?");
            actualRequiredShares--;
            console.log("Logging in with Visa Guide ID 1...");
          } else if (currentPriority.module === "socialProvider") {
            await this.tbsdk.modules.socialProvider.inputShareFromSecurityQuestions(this.answer2, "What's your Visa Guide ID?");
            requiredShares--;
            actualRequiredShares--;
            console.log("Logging in with Visa Guide ID 2 (aka Social Provider)...");
          } 

          if (tempSD.length === 0 && requiredShares > 0) {
            throw "URGENT: Need to Reassign Lost Key!";
          }
        }

        console.log(this.tbsdk);
        console.log(initializedDetails);
        if (actualRequiredShares > 0) {
          this.console('Not Enough Shares (Wrong Visa Guide ID or No Device Share Found');
          throw "Not Enough Shares";
        } else {
          let key = await this.tbsdk.reconstructKey();
          this.console("Logged In Successfully! Here's Your Private Key: " + JSON.stringify(key) + " Here's The Information For The Shares: " + JSON.stringify(shareDesc));
          console.log(JSON.stringify(key), this.tbsdk.getKeyDetails());
        }
      } catch (error) {
        console.error(error, "caught");
      }
    },
    async generateNewShare() {
      try {
        const res = await this.tbsdk.generateNewShare();
        console.log(res);
        this.console("New Share Info: " + JSON.stringify(res));
      } catch (error) {
        console.error(error, "GenerateNewShare() Error");
      }
    },
    async _initializeNewKey() {
      try {
        await this.initTkey();
        if (!this.mocked) await this.triggerLogin();
        const res = await this.tbsdk._initializeNewKey({ initializeModules: true });
        this.console("New tkey Info: " + JSON.stringify(res));
        console.log("New tkey Info:", res);
      } catch (error) {
        console.error(error, "initializeNewKey() Error");
      }
    },
    async initTkey() {
      const directParams = {
        baseUrl: `${location.origin}/serviceworker`,
        enableLogging: true,
        proxyContractAddress: "0x4023d2a0D330bF11426B12C6144Cfb96B7fa6183", // details for test net
        network: "testnet" // details for test net
      };
      const webStorageModule = new WebStorageModule();
      const visaGuide = new SecurityQuestionsModule();
      visaGuide.moduleName = "visaGuide";
      const socialProvider = new SecurityQuestionsModule();
      socialProvider.moduleName = "socialProvider";
      const shareTransferModule = new ShareTransferModule();

      let serviceProvider;
      if (this.mocked) {
        serviceProvider = new ServiceProviderBase({ postboxKey: "f1f02ee186749cfe1ef8f957fc3d7a5b7128f979bacc10ab3b2a811d4f990852" });
      } else {
        serviceProvider = new TorusServiceProvider({ directParams });
      }
      const storageLayer = new TorusStorageLayer({ hostUrl: "https://metadata.tor.us", serviceProvider: serviceProvider });
      const tbsdk = new ThresholdKey({
        serviceProvider: serviceProvider,
        storageLayer,
        modules: { webStorage: webStorageModule, visaGuide: visaGuide, socialProvider: socialProvider, shareTransfer: shareTransferModule }
      });
      this.tbsdk = tbsdk;
      this.torusdirectsdk = tbsdk.serviceProvider;

      if (!this.mocked) await tbsdk.serviceProvider.init({ skipSw: false });
    },
    console(text) {
      document.querySelector("#console>p").innerHTML = typeof text === "object" ? JSON.stringify(text) : text;
    }
  },

  async mounted() {
    try {
      await this.initTkey();
    } catch (error) {
      console.error(error, "mounted caught");
    }
  }
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
#console {
  border: 1px solid black;
  height: 80px;
  padding: 2px;
  bottom: 10px;
  position: absolute;
  text-align: left;
  width: calc(100% - 20px);
  border-radius: 5px;
}
#console::before {
  content: "Console :";
  position: absolute;
  top: -20px;
  font-size: 12px;
}
#console > p {
  margin: 0.5em;
  word-wrap: break-word;
}
#font-italic {
  font-style: italic;
}
button {
  margin: 0 10px 10px 0;
}
</style>