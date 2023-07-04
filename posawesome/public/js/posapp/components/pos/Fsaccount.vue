<template>
  
    <div>
      
      <v-autocomplete
        dense
        clearable
        auto-select-first
        outlined
        color="primary"
        :label="frappe._('Fs Account')"
        v-model="fs_number"
        :items="customers"
        item-text="fs_number"
        item-value="fs_number"
        background-color="white"
        :no-data-text="__('Customer not found')"
        hide-details
        :filter="customFilter"
        :disabled="readonly"
        append-icon="mdi-plus"
        @keyup.enter="onEnterKeyPress"
        @click:append="new_customer(fs_number  )"
        prepend-inner-icon="mdi-account-edit"
        @click:prepend-inner="edit_customer"
      >
        <template v-slot:item="data">
          <template>
            <v-list-item-content>
              <v-list-item-title
                class="primary--text subtitle-1"
                v-html="data.item.fs_number"
              ></v-list-item-title>
                                    
              
            </v-list-item-content>
          </template>
        </template>
      </v-autocomplete>
      <v-dialog v-model="insufficientFundsDialog" persistent max-width="300px">
        <v-card>
          <v-card-title class="headline">Insufficient Funds</v-card-title>
          <v-card-text>
            The account does not have sufficient funds to process this transaction.
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="green darken-1" text @click="processPending">Process Pending</v-btn>
            <v-btn color="red darken-1" text @click="insufficientFundsDialog = false">Cancel</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </div>
  </template>
  
  <script>
  import { evntBus } from '../../bus';
  
  export default {
    data: () => ({
      insufficientFundsDialog: false,
      pos_profile: '',
      customers: [],
      customer: '',
      fs_number: '',
      readonly: false,
    }),
  
    methods: {
      check_fs_balance(){
        let vm = this
        console.log("check fs banalnce"+vm.fs_number)
        //write frappe api call to this get_account_information
        // if the server is live set fs_mode to true else false
        frappe.call({
          method: 'frappe_fs.frappe_fs.doctype.fs_settings.fs_settings.has_positive_balance',
          args: {account_number: vm.fs_number},
          callback: function (r) {
            if (r.message) {
              console.log('Balance',r.message.data.data.has_positive_balance)
              evntBus.$emit('fs_postive_bc',r.message.data.data.has_positive_balance)
              if (!r.message.data.data.has_positive_balance) {
                vm.insufficientFundsDialog = true;
              }
          
            }
          },
        })
  
  
      },
      processPending() {
        // Code to process pending tasks
        evntBus.$emit('process_pending','Pending')
        this.insufficientFundsDialog = false;
      },
      onEnterKeyPress() {
      const foundCustomer = this.customers.find(
        (customer) => customer.fs_number === this.fs_number
      );
      if (foundCustomer) {
       console.log(foundCustomer)
       this.customer = foundCustomer.name
       evntBus.$emit('set_fs_number',this.fs_number)     
        this.check_fs_balance();
      } else {
        console.log('Customer not found', this.fs_number);
        this.new_customer(this.fs_number);
        // You can also handle the "not found" case as needed, e.g., show an alert or a notification
      }
      },
      getPOSProfileData() {
    const vm = this;
    return frappe
      .call('posawesome.posawesome.api.posapp.check_opening_shift', {
        user: frappe.session.user,
      })
      .then((r) => {
        if (r.message) {
          vm.pos_profile= r.message;
        } else {
          throw new Error("No POS profile data found");
        }
      });
  },
  
  get_customer_names() {
    const vm = this;
  
    const fetchCustomers = () => {
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
            console.info('loadCustomers');
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
    };
  
    if (!vm.pos_profile) {
      this.getPOSProfileData().then(fetchCustomers);
    } else {
      fetchCustomers();
    }
  },
      new_customer(fs_number) {
        evntBus.$emit('open_new_customer',fs_number);
      },
      edit_customer() {
        evntBus.$emit('open_edit_customer');
      },
      customFilter(item, queryText, itemText) {
        const fsNumberText = item.fs_number
    ? item.fs_number.toString().toLowerCase()
    : '';
  const searchText = queryText.toLowerCase();
  return fsNumberText.indexOf(searchText) > -1;
    }
    },
  
    computed: {},
    mounted: function () {
    this.get_customer_names();
  },
    created: function () {
      this.$nextTick(function () {
        evntBus.$on('reset_fs_number',(data)=>{
  
          this.fs_number ='';
  
        } );
        evntBus.$on('set_fs_number',(data)=>{
          this.fs_number = data
  
        })
        evntBus.$on('register_pos_profile', (pos_profile) => {
          this.pos_profile = pos_profile;
          this.get_customer_names();
        });
  
  
      });
    },
  
    watch: {
      customer(data) {
        evntBus.$emit('update_customer', data);
      },
      fs_number(data){
        evntBus.$emit('update_fs_number',data)
      }
    },
  };
  </script>