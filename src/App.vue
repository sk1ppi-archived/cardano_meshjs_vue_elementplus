<template>
    <el-container>
        <el-main>
            <!-- Wallet installed -->
            <!-- select wallet to use -->
            <el-select v-if="installedWallets" v-model="selectedWalletName" class="m-2"
                placeholder="Select wallet to connect" @change="selectWallet">
                <el-option v-for="wallet in installedWallets" :key="wallet.name" :label="wallet.name"
                    :value="wallet.name" />
            </el-select>

            <!-- Wallet selected -->
            <div v-if="!!selectedWallet">
                <el-divider></el-divider>

                <!-- Basic information about the wallet -->
                <el-descriptions direction="vertical" :column="4">
                    <el-descriptions-item label="Address" :span="4"> {{ selectedWalletAddress }}</el-descriptions-item>
                    <el-descriptions-item label="Balance" :span="4"> {{ selectedWalletBalance }} ADA</el-descriptions-item>
                    <el-descriptions-item label="Network" :span="4"> {{ selectedWalletNetwork }}</el-descriptions-item>
                </el-descriptions>

                <el-divider></el-divider>

                <el-form v-model="form" label-width="50px">
                    <el-form-item label="Name">
                        <el-input v-model="form.assetName" placeholder="Name" />
                    </el-form-item>

                    <el-form-item label="Amount">
                        <el-input v-model="form.assetQuantity" type="number" placeholder="Quantity" />
                    </el-form-item>

                    <el-form-item label="Label">
                        <el-select v-model="form.assetLabel">
                            <el-option label="Non Fungible Token (NFT)" value="721"></el-option>
                            <el-option label="Fungible Token" value="20"></el-option>
                        </el-select>
                    </el-form-item>


                    <el-form-item label="Metadata">
                        <el-input v-model="form.assetMetadata" type="textarea" />
                    </el-form-item>

                    <el-form-item>
                        <el-button type="primary" @click="onSubmit">Create</el-button>
                    </el-form-item>

                </el-form>
            </div>
        </el-main>
    </el-container>
</template>

<script>
import { BrowserWallet, BlockfrostProvider, Transaction, ForgeScript } from '@meshsdk/core';

const BLOCKFROST_API_KEY = ''

export default {
    name: 'App',

    data: () => ({
        form: {

            assetName: "XYZ token",
            assetQuantity: 1,
            assetLabel: '721',

            assetMetadata: JSON.stringify({
                name: "XYZ token",
                image: "ipfs://QmRzicpReutwCkM6aotuKjErFCUD213DpwPq6ByuzMJaua",
                mediaType: "image/jpg",
                description: "This NFT is minted by Mesh (https://meshjs.dev/).",
            })
        },

        installedWallets: [],

        selectedWalletName: '',
        selectedWallet: false,
        selectedWalletAddress: '',
        selectedWalletBalance: '',
        selectedWalletNetwork: '',

        blockchainProvider: new BlockfrostProvider(BLOCKFROST_API_KEY)
    }),

    async mounted() {
        // Connect wallet
        this.installedWallets = await BrowserWallet.getInstalledWallets()
    },


    methods: {

        // Select wallet to connect
        async selectWallet(walletName) {
            this.selectedWallet = await BrowserWallet.enable(walletName)

            await this.updateBalance(this.selectedWallet)
            await this.updateAddress(this.selectedWallet)
            await this.updateNetwork(this.selectedWallet)
        },


        // Balance of selected wallet
        async updateBalance(wallet) {
            this.selectedWalletBalance = await wallet.getBalance()
                .then(units => units
                    .filter(u => u.unit == 'lovelace')
                    .map(u => parseInt(u.quantity))
                    .reduce((a, b) => a + b, 0) / 1000000
                )
        },

        // Used address of selected wallet
        async updateAddress(wallet) {
            this.selectedWalletAddress = await wallet.getUsedAddresses()
                .then(addresses => addresses.length ? addresses[0] : addresses[addresses.length - 1])
        },

        // Network of selected wallet
        async updateNetwork(wallet) {
            this.selectedWalletNetwork = await wallet.getNetworkId()
                .then(id => id === 0 ? 'testnet' : 'mainnet')
        },

        // Handle form submission to create token
        async onSubmit() {

            const wallet = this.selectedWallet
            const tx = new Transaction({ initiator: wallet });

            const address = this.selectedWalletAddress

            // prepare forgingScript
            const forgingScript = ForgeScript.withOneSignature(address);

            const asset = {
                assetName: this.form.assetName,
                assetQuantity: String(this.form.assetQuantity),
                metadata: JSON.parse(this.form.assetMetadata),
                label: this.form.assetLabel,
                recipient: address
            }

            tx.mintAsset(forgingScript, asset);

            const unsignedTx = await tx.build();
            const signedTx = await wallet.signTx(unsignedTx);
            const txHash = await wallet.submitTx(signedTx);

            alert(txHash)
        }
    }
}
</script>