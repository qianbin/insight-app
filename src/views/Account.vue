<template>
    <b-container>
        <b-card no-body>
            <b-card-header :class="{'border-bottom-0':account,'pb-0':account}">
                <span class="h4 mr-3">Account</span>

                <AccountLink no-link icon :address="address"/>
                <VeForgeLink btn type="acc" :arg="address" class="float-right"/>
            </b-card-header>
            <b-tabs v-if="account" card no-key-nav v-model="tab">
                <b-tab title="Summary">
                    <div>
                        <b-badge
                            :variant="account.hasCode? 'success':'secondary'"
                        >{{account.hasCode? 'Contract': 'Regular'}}</b-badge>
                    </div>
                    <hr>
                    <b-row>
                        <b-col lg="3">
                            <strong>Balance</strong>
                        </b-col>
                        <b-col lg="9">
                            <Amount sym="VET">{{account.balance}}</Amount>
                        </b-col>
                    </b-row>
                    <hr>
                    <b-row>
                        <b-col lg="3">
                            <strong>Energy</strong>
                        </b-col>
                        <b-col lg="9">
                            <Amount sym="VTHO">{{account.energy}}</Amount>
                        </b-col>
                    </b-row>
                    <template v-if="account.hasCode">
                        <hr>
                        <b-row>
                            <b-col lg="3">
                                <strong>Runtime Bytecode</strong>
                            </b-col>
                            <b-col lg="9">
                                <b-textarea
                                    v-if="code"
                                    class="text-monospace bg-light"
                                    size="sm"
                                    readonly
                                    :value="code.code"
                                />
                                <template v-else>
                                    <b-button size="sm" @click="loadCode" :disabled="loadingCode">
                                        View Code
                                        <b-spinner v-if="loadingCode" type="grow" small/>
                                    </b-button>
                                    <span
                                        v-if="codeError"
                                        class="text-warning ml-3"
                                    >codeError.message</span>
                                </template>
                            </b-col>
                        </b-row>
                    </template>
                </b-tab>
                <b-tab title="Transfers" @click="loadTransfers">
                    <p>
                        <b-button
                            class="px-3 mr-2"
                            @click="loadTransfers"
                            size="sm"
                            :disabled="transfers.loading"
                            variant="outline-secondary"
                        >⟳</b-button>
                        <b-button-group size="sm" :disabled="transfers.loading">
                            <b-button
                                class="px-3"
                                :disabled="!transfers.canPrev"
                                @click="transfersPrevPage"
                                variant="outline-secondary"
                            >&lsaquo;</b-button>
                            <b-button
                                class="px-3"
                                :disabled="!transfers.canNext"
                                @click="transfersNextPage"
                                variant="outline-secondary"
                            >&rsaquo;</b-button>
                        </b-button-group>
                        <span
                            v-if="transfers.range"
                            class="ml-3"
                        >{{transfers.range[0]}} - {{transfers.range[1]}}</span>
                    </p>
                    <template v-if="transfers.created">
                        <Loading v-if="transfers.loading" class="my-3"/>
                        <div v-else-if="transfers.error" class="text-center">
                            <p>Oops</p>
                            <p class="text-warning">Error: {{transfers.error.message}}</p>
                            <b-button size="sm" @click="reload">Reload</b-button>
                        </div>
                        <b-list-group flush v-else-if="transfers.items.length">
                            <b-list-group-item
                                v-for="(item,i) in transfers.items"
                                :key="i"
                                :to="{name: 'tx', params:{id: item.meta.txID}}"
                            >
                                <Transfer :item="item" :index="i" :owner="address"/>
                            </b-list-group-item>
                        </b-list-group>
                        <div v-else class="text-center">No content</div>
                    </template>
                </b-tab>
                <b-tab title="Events" @click="loadEvents">
                    <p>
                        <b-button
                            class="px-3 mr-2"
                            @click="loadEvents"
                            size="sm"
                            :disabled="events.loading"
                            variant="outline-secondary"
                        >⟳</b-button>
                        <b-button-group size="sm" :disabled="events.loading">
                            <b-button
                                class="px-3"
                                :disabled="!events.canPrev"
                                @click="eventsPrevPage"
                                variant="outline-secondary"
                            >&lsaquo;</b-button>
                            <b-button
                                class="px-3"
                                :disabled="!events.canNext"
                                @click="eventsNextPage"
                                variant="outline-secondary"
                            >&rsaquo;</b-button>
                        </b-button-group>
                        <span
                            v-if="events.range"
                            class="ml-3"
                        >{{events.range[0]}} - {{events.range[1]}}</span>
                    </p>
                    <template v-if="events.created">
                        <Loading v-if="events.loading" class="my-3"/>
                        <div v-else-if="events.error" class="text-center">
                            <p>Oops</p>
                            <p class="text-warning">Error: {{events.error.message}}</p>
                            <b-button size="sm" @click="reload">Reload</b-button>
                        </div>
                        <template v-else-if="events.items.length">
                            <Event
                                v-for="(item,i) in events.items"
                                :key="i"
                                :item="item"
                                :index="i + events.offset"
                                :class="{'mt-3':i>0}"
                                class="small"
                            />
                        </template>
                        <div v-else class="text-center">No content</div>
                    </template>
                </b-tab>
                <b-tab title="Deposit">
                    <Deposit :address="address" style="width:30rem;" class="mx-auto"/>
                </b-tab>
            </b-tabs>
            <template v-else>
                <div v-if="error" class="text-center">
                    <p>Oops</p>
                    <p class="text-warning">Error: {{error.message}}</p>
                    <b-button size="sm" @click="reload">Reload</b-button>
                </div>
                <Loading v-else class="my-3"/>
            </template>
        </b-card>
    </b-container>
</template>
<script lang="ts">
import { Vue, Component, Watch } from 'vue-property-decorator'

@Component
export default class Account extends Vue {
    private tab = 0
    private address = ''

    private account: Connex.Thor.Account | null = null
    private error: Error | null = null

    private code: Connex.Thor.Code | null = null
    private codeError: Error | null = null
    private loadingCode = false

    private events = {
        items: null as Connex.Thor.Event[] | null,
        error: null as Error | null,
        loading: false,
        offset: 0,
        created: false,
        get canNext() { return this.items && this.items.length === 10 },
        get canPrev() { return this.items && this.offset > 0 },
        get range() {
            if (!this.loading && this.items && this.items.length > 0) {
                return [this.offset, this.offset + this.items.length]
            }
            return null
        }
    }
    private transfers = {
        items: null as Connex.Thor.Transfer[] | null,
        error: null as Error | null,
        loading: false,
        offset: 0,
        created: false,
        get canNext() { return this.items && this.items.length === 10 },
        get canPrev() { return this.items && this.offset > 0 },
        get range() {
            if (!this.loading && this.items && this.items.length > 0) {
                return [this.offset, this.offset + this.items.length]
            }
            return null
        }
    }

    @Watch('$store.state.chainStatus')
    private async reload() {
        try {
            const acc = await connex.thor.account(this.address).get()
            this.account = acc
        } catch (err) {
            this.error = err
        }
    }

    private async loadCode() {
        try {
            this.loadingCode = true
            const code = await connex.thor.account(this.address).getCode()
            this.code = code
        } catch (err) {
            this.codeError = err
        } finally {
            this.loadingCode = false
        }
    }

    private async loadEvents() {
        this.events.created = true
        if (this.events.loading) {
            return
        }
        try {
            this.events.loading = true
            this.events.error = null
            this.events.items = await connex.thor.filter('event')
                .criteria([{ address: this.address }])
                .order('desc')
                .apply(this.events.offset, 10)
        } catch (err) {
            this.events.error = err
        } finally {
            this.events.loading = false
        }
    }
    private async loadTransfers() {
        this.transfers.created = true
        if (this.transfers.loading) {
            return
        }
        this.transfers.error = null
        this.transfers.loading = true
        this.transfers.items = null
        try {
            this.transfers.items = await connex.thor.filter('transfer')
                .criteria([{ sender: this.address }, { recipient: this.address }])
                .order('desc')
                .apply(this.transfers.offset, 10)
        } catch (err) {
            this.transfers.error = err
        } finally {
            this.transfers.loading = false
        }
    }

    private async transfersNextPage() {
        this.transfers.offset += 10
        this.loadTransfers()
    }

    private async transfersPrevPage() {
        if (this.transfers.offset >= 10) {
            this.transfers.offset -= 10
            this.loadTransfers()
        }
    }

    private async eventsNextPage() {
        this.events.offset += 10
        this.loadEvents()
    }

    private async eventsPrevPage() {
        if (this.events.offset >= 10) {
            this.events.offset -= 10
            this.loadEvents()
        }
    }

    private created() {
        this.$ga.page('/insight/account')
        this.address = this.$route.params.address.toLowerCase()
        this.reload()
    }
}

</script>
