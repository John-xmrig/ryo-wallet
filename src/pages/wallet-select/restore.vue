<template>
<q-page>
    <div class="q-mx-md">
        <q-field class="q-mt-none">
            <q-input
                v-model="wallet.name"
                float-label="Wallet name"
                @blur="$v.wallet.name.$touch"
                :error="$v.wallet.name.$error"
                :dark="theme=='dark'"
                />
        </q-field>

        <q-field>
            <q-input
                v-model="wallet.seed"
                float-label="Mnemonic seed"
                type="textarea"
                @blur="$v.wallet.seed.$touch"
                :error="$v.wallet.seed.$error"
                :dark="theme=='dark'"
                />
        </q-field>

        <q-field>
            <div class="row items-center gutter-sm">
                <div class="col">
                    <template v-if="wallet.refresh_type=='date'">
                        <q-datetime v-model="wallet.refresh_start_date" type="date"
                                    float-label="Restore from date"
                                    modal :min="1492486495000" :max="Date.now()"
                                    :dark="theme=='dark'"
                                    />
                    </template>
                    <template v-else-if="wallet.refresh_type=='height'">
                        <q-input v-model="wallet.refresh_start_height" type="number"
                                 min="0" float-label="Restore from block height"
                                 @blur="$v.wallet.refresh_start_height.$touch"
                                 :error="$v.wallet.refresh_start_height.$error"
                                 :dark="theme=='dark'"
                                 />
                    </template>
                </div>
                <div class="col-auto self-end">
                    <template v-if="wallet.refresh_type=='date'">
                        <q-btn @click="wallet.refresh_type='height'" class="float-right" :text-color="theme=='dark'?'white':'dark'" flat>
                            <div style="width: 80px;" class="text-center">
                                <q-icon class="block" name="clear_all" />
                                <div style="font-size:10px">Switch to<br/>height select</div>
                            </div>
                        </q-btn>
                    </template>
                    <template v-else-if="wallet.refresh_type=='height'">
                        <q-btn @click="wallet.refresh_type='date'" class="float-right" :text-color="theme=='dark'?'white':'dark'" flat>
                            <div style="width: 80px;" class="text-center">
                                <q-icon class="block" name="today" />
                                <div style="font-size:10px">Switch to<br/>date select</div>
                            </div>
                        </q-btn>
                    </template>
                </div>
            </div>
        </q-field>

        <q-field>
            <q-input v-model="wallet.password" type="password" float-label="Password" :dark="theme=='dark'" />
        </q-field>

        <q-field>
            <q-input v-model="wallet.password_confirm" type="password" float-label="Confirm Password" :dark="theme=='dark'" />
        </q-field>

        <PasswordStrength :password="wallet.password" ref="password_strength" />

        <q-field>
            <q-btn color="primary" @click="restore_wallet" label="Restore wallet" />
        </q-field>

    </div>

    <WalletLoading ref="loading" />

</q-page>
</template>

<script>
import PasswordStrength from "components/password_strength"
import WalletLoading from "components/wallet_loading"
import { required, numeric } from "vuelidate/lib/validators"
import { mapState } from "vuex"
export default {
    data () {
        return {
            wallet: {
                name: "",
                seed: "",
                refresh_type: "date",
                refresh_start_height: 0,
                refresh_start_date: 1492486495000, // timestamp of block 1
                password: "",
                password_confirm: ""
            },
        }
    },
    computed: mapState({
        notify_empty_password: state => state.gateway.app.config.preference.notify_empty_password,
        theme: state => state.gateway.app.config.appearance.theme,
        status: state => state.gateway.wallet.status,
    }),
    watch: {
        status: {
            handler(val, old){
                if(val.code == old.code) return
                switch(this.status.code) {
                    case 1:
                        break;
                    case 0:
                        this.$refs.loading.hide()
                        this.$router.replace({ path: "/wallet-select/created" });
                        break;
                    default:
                        this.$refs.loading.hide()
                        this.$q.notify({
                            type: "negative",
                            timeout: 1000,
                            message: this.status.message
                        })
                        break;
                }
            },
            deep: true
        }
    },
    validations: {
        wallet: {
            name: { required },
            seed: { required },
            refresh_start_height: { numeric }
        }
    },
    methods: {
        restore_wallet() {
            this.$v.wallet.$touch()

            if (this.$v.wallet.name.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: "Enter a wallet name"
                })
                return
            }
            if (this.$v.wallet.seed.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: "Enter seed words"
                })
                return
            }

            let seed = this.wallet.seed.trim().replace(/\s{2,}/g, " ").split(" ")
            if(seed.length !== 14 && seed.length !== 24 && seed.length !== 25 && seed.length !== 26) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: "Invalid seed word length"
                })
                return
            }

            if (this.$v.wallet.refresh_start_height.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: "Invalid restore height"
                })
                return
            }
            if(this.wallet.password != this.wallet.password_confirm) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: "Passwords do not match"
                })
                return
            }

            this.warnEmptyPassword()
                .then(options => {
                    if(options.length > 0 && options[0] === true) {
                        // user selected do not show again
                        this.$gateway.send("core", "quick_save_config", {
                            preference: {
                                notify_empty_password: false
                            }
                        })
                    }

                    this.$refs.loading.show()

                    this.$gateway.send("wallet", "restore_wallet", this.wallet);

                }).catch(() => {
                })
        },
        cancel() {
            this.$router.replace({ path: "/wallet-select" });
        },
        warnEmptyPassword: function () {
            let message = ""
            if(this.wallet.password == "") {
                message = "Using an empty password will leave your wallet unencrypted on your file system!"
            } else if(this.$refs.password_strength.score < 3) {
                message = "Using an insecure password could allow attackers to brute-force your wallet! Consider using a password with better strength."
            }
            if(this.notify_empty_password && message != "") {
                return this.$q.dialog({
                    title: "Warning",
                    message: message,
                    options: {
                        type: "checkbox",
                        model: [],
                        items: [
                            {label: "Do not show this message again", value: true},
                        ]
                    },
                    ok: {
                        label: "CONTINUE"
                    },
                    cancel: {
                        flat: true,
                        label: "CANCEL",
                        color: this.theme=="dark"?"white":"dark"
                    }
                })
            } else {
                return new Promise((resolve, reject) => {
                    resolve([])
                })
            }
        }
    },
    components: {
        PasswordStrength,
        WalletLoading
    }
}
</script>

<style>
</style>
