<template>
    <div>
        <!-- app bar -->
        <div>
            <v-row class="ml-1 mt-1">
                <v-col cols="2">
                    <v-text-field rounded outlined dense label="Qidirish" @keyup.enter="fetchCosts()" v-model="search">
                    </v-text-field>
                </v-col>
                <v-col cols="2">
                    <v-menu v-model="menu" :close-on-content-click="false" :nudge-right="40" transition="scale-transition"
                        offset-y>
                        <template v-slot:activator="{ on }">
                            <v-text-field v-model="selectedRange" :append-icon="menu ? 'mdi-calendar' : ''" outlined dense
                                placeholder="Vaqt oralig'ini tanlang..." :clearable="!menu" readonly
                                @click:append="fetchCosts(menu)" v-on="on"></v-text-field>
                        </template>
                        <v-date-picker v-model="selectedRange" range
                            @input="selectedRange.length == 2 ? selectCalendar() : null"></v-date-picker>
                    </v-menu>
                </v-col>
                <v-spacer></v-spacer>
                <v-col cols="2">
                    <v-btn color="success" @click="loadExcel('cost/load_table_excel')">
                        <v-icon>
                            mdi-microsoft-excel
                        </v-icon>
                        Oynani Excel ga yuklash
                    </v-btn>
                </v-col>
                <v-col cols="1" class="mr-6">
                    <v-btn color="success" @click="openDialog()">
                        <v-icon>
                            mdi-plus-circle
                        </v-icon>
                        Qo`shish
                    </v-btn>
                </v-col>
            </v-row>
        </div>
        <!-- Cancel dialog -->
        <v-col cols="auto" v-if="cancelDialog">
            <v-dialog transition="dialog-top-transition" max-width="600" v-model="cancelDialog">
                <v-card>
                    <v-card-title style="background-color: #499C54; color: beige">№ {{ cost.id }}-xarajatni bekor qilasizmi ?</v-card-title>
                    <v-card-text>
                        <v-text-field
                            label="Izox"
                            v-model="cost.description"
                        ></v-text-field>
                    </v-card-text>
                    <v-divider></v-divider>
                    <v-card-actions class="justify-end">
                        <v-btn text color="primary" @click="cancelCost()">Tasdiqlash</v-btn>
                        <v-btn text color="orange" @click="cancelDialog = false">Berkitish</v-btn>
                    </v-card-actions>
                </v-card>
            </v-dialog>
        </v-col>
        <!-- Excel dialog -->
        <v-col cols="auto" v-if="excelDialog">
            <v-dialog transition="dialog-top-transition" max-width="600" v-model="excelDialog">
                <v-card>
                    <v-toolbar style="text-align: center;" color="primary" dark class="text-h5 mb-4 pa-1">Ma'lumot yuklashga
                        tayyor</v-toolbar>
                    <v-divider></v-divider>
                    <v-card-actions class="justify-end">
                        <v-btn text color="primary" @click="downloadExcelFile()">Yuklash</v-btn>
                        <v-btn text color="orange" @click="excelDialog = false">Berkitish</v-btn>
                    </v-card-actions>
                </v-card>
            </v-dialog>
        </v-col>
        <!-- pay dialog -->
        <v-col cols="auto">
            <v-dialog transition="dialog-top-transition" max-width="600" v-model="payDialog">
                <v-card>
                    <v-toolbar color="primary" dark class="text-h4 mb-4">To'lov qilish #id: {{ cost.id }}</v-toolbar>
                    <v-card-text>
                        <v-text-field outlined required label="Qiymat" v-model="cost.amount" @keypress="filter()" />
                    </v-card-text>
                    <v-divider></v-divider>
                    <v-card-actions class="justify-end">
                        <v-checkbox v-if="auth.status == 'admin'" label="Perechesleniya" v-model="cost.is_cashflow"
                            :value="cost.is_cashflow"></v-checkbox>
                        <v-spacer></v-spacer>
                        <v-btn text color="primary" @click="payThrCost()">To'lash</v-btn>
                        <v-btn text color="red" @click="payDialog = false">Berkitish</v-btn>
                    </v-card-actions>
                </v-card>
            </v-dialog>
        </v-col>
        <!-- add dialog -->
        <v-col cols="auto">
            <v-dialog transition="dialog-top-transition" max-width="600" v-model="dialog">
                <v-card>
                    <v-toolbar color="primary" dark class="text-h4 mb-4">{{ action == 'update' ? 'Yangilash' : "Qo'shish"
                    }}</v-toolbar>
                    <v-card-text>
                        <v-autocomplete v-model="cost.cost_type_id" :items="activeCostTypes" item-value="id"
                            item-text="title" :clearable="true" outlined label="Xarajat turi" />
                        <v-autocomplete v-model="cost.location_id" :items="locations" item-value="id" item-text="title"
                            :clearable="true" outlined label="Xudud" />
                        <v-text-field outlined required label="Qiymat" v-model="cost.amount" @keypress="filter()" />
                        <v-text-field outlined required label="Izoh" v-model="cost.description" />
                    </v-card-text>
                    <v-divider></v-divider>
                    <v-card-actions class="justify-end">
                        <v-btn text color="primary" @click="save()">Saqlash</v-btn>
                        <v-btn text color="red" @click="dialog = false">Berkitish</v-btn>
                    </v-card-actions>
                </v-card>
            </v-dialog>
        </v-col>
        <!-- table -->
        <v-data-table hide-default-footer :headers="headers" :items="costs" dense class="elevation-1" item-key="id"
            loading="true" disable-sort>
            <template v-slot:item.amount="{ item }">
                <span>{{ formatMoney.format(item.amount) }}</span>
            </template>
            <template v-slot:item.payed="{ item }">
                <span>{{ formatMoney.format(item.payed) }}</span>
            </template>
            <template v-slot:item.actions="{ item }">
                <!-- <v-btn v-if="auth.status == 'admin'" @click="openDialog(item)" color="success" dense icon rounded>
                    <v-icon>mdi-border-color</v-icon>
                </v-btn> -->
                <v-tooltip top v-if="auth.status == 'admin' && item.status != 'Bekor qilindi'">
                    <template #activator="{ on, attrs }">
                        <v-btn @click="openCancelDialog(item)" color="error" dense icon rounded v-on="on" v-bind="attrs">
                            <v-icon>mdi-close-circle-multiple-outline</v-icon>
                        </v-btn>
                    </template>
                    <span>Bekor qilish</span>
                </v-tooltip>
                <v-tooltip top v-if="item.pay_btn">
                    <template #activator="{ on, attrs }">
                        <v-btn @click="openPayDialog(item)" color="success" dense icon rounded v-on="on" v-bind="attrs">
                            <v-icon>mdi-send-clock</v-icon>
                        </v-btn>
                    </template>
                    <span>Xarajat pulini to'lash</span>
                </v-tooltip>
            </template>
        </v-data-table>
        <v-pagination :length="last_page" v-model="page" total-visible="8"></v-pagination>
    </div>
</template>
<script>
import axios from 'axios'
import { mapGetters } from 'vuex'
export default {
    name: 'Cost',
    watch: {
        page() {
            this.fetchCosts()
        },
        selectedRange() {
            if (!this.selectedRange) {
                this.fetchCosts()
            }
        }
    },
    data() {
        return {
            search: '',
            page: 1,
            last_page: 1,
            auth: JSON.parse(localStorage.getItem('user')),
            dialog: false,
            payDialog: false,
            cancelDialog: false,
            action: 'update',
            cost: {},
            selectedRange: null,
            menu: false,
            excelUrl: '',
            formatMoney: new Intl.NumberFormat('uz-UZ', {
                style: 'currency',
                currency: 'UZS',
            }),
            excelDialog: false,
            headers: [
                { text: '#', value: 'id' },
                { text: 'Bajardi', value: 'auth.full_name' },
                { text: 'Xarajat kodi', value: 'cost_type.code' },
                { text: 'Xarajat', value: 'cost_type.title' },
                { text: 'Xudud', value: 'location.title' },
                { text: 'Umumiy summa', value: 'amount' },
                { text: 'To\'landi', value: 'payed' },
                { text: 'Izoh', value: 'description' },
                { text: 'Sana', value: 'date' },
                { text: 'Status', value: 'status' },
                { text: 'Amallar', value: 'actions' },
            ]
        }
    },
    computed: {
        ...mapGetters({
            costs: 'cost/getCosts',
            activeCostTypes: 'costType/getActiveCostTypes',
            locations: 'location/getActiveLocations',
        })
    },
    async created() {
        await this.fetchCosts()
        await this.fetchActiveCostTypes()
        await this.fetchActiveLocations()
    },
    methods: {
        async fetchCosts(menu) {
            try {
                const { data } = await this.$store.dispatch('cost/fetchCosts', {
                    search: this.search,
                    page: this.page,
                    dates: this.selectedRange
                })
                this.last_page = data.meta.last_page
            } catch (error) {
                this.$toast.error(error.response.data.message)
            }
        },
        async fetchActiveCostTypes() {
            try {
                await this.$store.dispatch('costType/fetchActiveCostTypes')
            } catch (error) {
                this.$toast.error(error.response.data.message)
            }
        },
        async fetchActiveLocations() {
            try {
                await this.$store.dispatch('location/fetchActiveLocations')
            } catch (error) {
                this.$toast.error(error.response.data.message)
            }
        },
        openDialog(item = null) {
            this.dialog = true
            if (item == null) {
                this.action = 'create'
            }
            this.cost = item ?? {}
        },
        openCancelDialog(item) {
            this.cancelDialog = true
            let cost = {...item}
            cost.description = null
            this.cost = cost
        },
        save() {
            if (this.action == 'update') {

            } else {
                this.createCost()
            }
        },
        async selectCalendar() {
            if (this.selectedRange.length == 2) {
                let firstDay = this.selectedRange[0];
                let presentDay = this.selectedRange[1];
                if (firstDay > presentDay) {
                    this.selectedRange[0] = presentDay;
                    this.selectedRange[1] = firstDay;
                }
                this.fetchCosts()
                this.menu = false
            }
        },
        async createCost() {
            try {
                const res = await this.$store.dispatch('cost/createCost', this.cost)
                this.dialog = false
                await this.fetchCosts()
                this.$toast.success(res.data.message)
            } catch (error) {
                this.$toast.error(error.response.data.message)
            }
        },
        filter() {
            const evt = window.event;
            let expect = evt.target.value.toString() + evt.key.toString();

            if (!/^[-+]?[0-9]*\.?[0-9]*$/.test(expect)) {
                evt.preventDefault();
            } else {
                return true;
            }
        },
        openPayDialog(cost) {
            this.cost = cost
            this.payDialog = true
        },
        async payThrCost() {
            try {
                const res = await this.$store.dispatch('cost/payTheCost', {
                    id: this.cost.id,
                    amount: this.cost.amount,
                    is_cashflow: this.cost.is_cashflow
                })
                await this.fetchCosts()
                this.payDialog = false
                this.$toast.success(res.data.message)
            } catch (error) {
                this.$toast.error(error.response.data.message)
            }
        },
        async cancelCost(){
            try {
                const res = await this.$store.dispatch('cost/cancelCost', {
                    id: this.cost.id,
                    comment: this.cost.description
                })
                await this.fetchCosts()
                this.cancelDialog = false
                this.$toast.success(res.data.message)
            } catch (error) {
                this.$toast.error(error.response.data.message)
            }
        },
        async loadExcel(api) {
            try {
                const res = await this.$store.dispatch('cost/loadExcel', {
                    api,
                    dates: this.selectedRange,
                    search: this.search
                })
                this.excelUrl = res.data.url_
                this.excelDialog = true
            } catch (error) {
                this.$toast.error(error.response.data.message)
            }
        },
        async downloadExcelFile() {
            try {
                const fileUrl = process.env.VUE_APP_NATIVE_URL + this.excelUrl;
                window.location.href = fileUrl
                this.excelDialog = false;
            } catch (error) {
                this.$toast.error(error.response.data.message)
            }
        }
    }
}
</script>