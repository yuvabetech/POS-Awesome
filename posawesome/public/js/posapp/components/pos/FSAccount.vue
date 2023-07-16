<template>
    <div>
      <v-autocomplete
        dense
        clearable
        auto-select-first
        outlined
        color="primary"
        :label="frappe._('Account Number')"
        v-model="customer"
        :items="customers"
        item-text="fs_account_number"
        item-value="name"
        background-color="white"
        :no-data-text="__('Account Number not found')"
        hide-details
        :filter="customFilter"
        :disabled="readonly"
      >
        <template v-slot:item="data">
          <template>
            <v-list-item-content>
              <v-list-item-title
                class="primary--text subtitle-1"
                v-html="data.item.fs_account_number"
              ></v-list-item-title>
            </v-list-item-content>
          </template>
        </template>
      </v-autocomplete>
      <div class="mb-8">
        <UpdateCustomer></UpdateCustomer>
      </div>
    </div>
  </template>
  
  <script>
  import { evntBus } from '../../bus';
  import UpdateCustomer from './UpdateCustomer.vue';
  export default {
    data: () => ({
      pos_profile: '',
      customers: [],
      customer: '',
      readonly: false,
      customer_info: {},
    }),
  
    components: {
      UpdateCustomer,
    },
  
    methods: {
      get_customer_names() {
        const vm = this;
        if (this.customers.length > 0) {
          return;
        }
        if (vm.pos_profile.posa_local_storage && localStorage.customer_storage) {
          vm.customers = JSON.parse(localStorage.getItem('customer_storage'));
        }
        frappe.call({
          method: 'posawesome.posawesome.api.posapp.get_customer_names',
          args: {
            pos_profile: this.pos_profile.pos_profile,
          },
          callback: function (r) {
            if (r.message) {
              vm.customers = r.message;
              if (vm.pos_profile.posa_local_storage) {
                localStorage.setItem('customer_storage', '');
                localStorage.setItem(
                  'customer_storage',
                  JSON.stringify(r.message)
                );
              }
            }
          },
        });
      },
      customFilter(item, queryText, itemText) {
        const textOne = item.fs_account_number
          ? item.fs_account_number
          : '';
        const searchText = queryText
  
        return (
          textOne.indexOf(searchText) > -1
        );
      },
    },
  
    computed: {},
  
    created: function () {
      
      this.$nextTick(function () {
        evntBus.$on('register_pos_profile', (pos_profile) => {
          this.pos_profile = pos_profile;
          this.get_customer_names();
        });
        evntBus.$on('payments_register_pos_profile', (pos_profile) => {
          this.pos_profile = pos_profile;
          this.get_customer_names();
        });
        evntBus.$on('set_customer', (customer) => {
          this.customer = customer;
        });
        evntBus.$on('add_customer_to_list', (customer) => {
          this.customers.push(customer);
        });
        evntBus.$on('set_customer_readonly', (value) => {
          this.readonly = value;
        });
        evntBus.$on('set_customer_info_to_edit', (data) => {
          this.customer_info = data;
        });
        evntBus.$on('fetch_customer_details', () => {
          this.get_customer_names();
        });
      });
    },
  
    watch: {
      customer() {
        evntBus.$emit('update_customer', this.customer);
      },
    },
  };
  </script>
  