<template>
    <div>
      <div class="add-circle" @click="openNewShockerModal">
        <i class="fa-solid fa-plus"></i>
      </div>

    <loading v-if="loadingShockers"/>

    <b-container>
        <b-row v-for="item in ownShockers" :key="item.id">
            <device :device="item"></device>
        </b-row>
    </b-container>

    <b-modal v-model="modal.new" title="New Shocker" ok-title="Create" @ok.prevent="createNewShocker">
        <loading v-if="devicesLoading"></loading>
        <div v-else>
          <b-container style="padding: 0;">
            <b-form-group label="Shocker's device" label-for="device">
                <b-form-select :state="validationDevice" id="device" v-model="selectedDevice" :options="deviceList" required/>
                <b-form-invalid-feedback :state="validationDevice">
                    Select a hub
                </b-form-invalid-feedback>
            </b-form-group>

            <b-form-group label="Name" label-for="item-name" label-class="mb-1">
              <b-form-input id="item-name" v-model="newName" :state="validationName"/>
              <b-form-invalid-feedback :state="validationName">
                    Must be between 1 and 48 characters
                </b-form-invalid-feedback>
            </b-form-group>

            <b-form-group label="Rf Id" label-for="item-rf" label-class="mb-1">
              <b-form-input id="item-rf" v-model="newRfId" type="number" :state="validationRfId"/>
              <b-form-invalid-feedback :state="validationRfId">
                    Must be a number between 0 - 65535
                </b-form-invalid-feedback>
            </b-form-group>

            <b-form-group label="Model" label-for="item-model" label-class="mb-1">
              <b-form-select id="item-model" :state="validationModel" v-model="newModel" :options="modelOptions"/>
              <b-form-invalid-feedback :state="validationModel">
                    Select a model type
                </b-form-invalid-feedback>
            </b-form-group>
          </b-container>
        </div>
      </b-modal>

      <b-modal v-model="modal.edit" title="Edit Shocker" ok-title="Save" @ok.prevent="applyEdits">
        <loading v-if="modal.editLoading"></loading>
        <div v-else>
          <b-container style="padding: 0;">
            <b-form-group label="ID" label-for="item-id" label-class="mb-1">
              <b-form-input id="item-id" v-model="retrievedShocker.id" readonly/>
            </b-form-group>

            <b-form-group label="Hub" label-for="item-device" label-class="mb-1">
              <b-form-select id="item-device" :state="validationEditDevice" v-model="retrievedShocker.device" :options="deviceList" required/>
              <b-form-invalid-feedback :state="validationEditDevice">
                    Select a hub
                </b-form-invalid-feedback>
            </b-form-group>
        
            <b-form-group label="Name" label-for="item-name" label-class="mb-1">
              <b-form-input id="item-name" :state="validationEditName" v-model="retrievedShocker.name"/>
              <b-form-invalid-feedback :state="validationEditName">
                    Must be between 1 and 48 characters
                </b-form-invalid-feedback>
            </b-form-group>
            
            <b-form-group label="RfId" label-for="item-rfid" label-class="mb-1">
              <b-form-input id="item-rfid" :state="validationEditRfId" v-model="retrievedShocker.rfId"/>
              <b-form-invalid-feedback :state="validationEditRfId">
                    Must be a number between 0 - 65535
                </b-form-invalid-feedback>
            </b-form-group>

            <b-form-group label="Model" label-for="item-model" label-class="mb-1">
              <b-form-select id="item-model" v-model="retrievedShocker.model" :options="modelOptions"/>
            </b-form-group>
          </b-container>
        </div>
      </b-modal>
    </div>
</template>

<script>
import Loading from '../../../utils/Loading.vue';
import Device from './Device.vue';

export default {
    components: {Loading, Device},
    data() {
        return {
            loadingShockers: true,
            ownShockers: [],
            devices: [],
            modelOptions: [
                {
                    text: "CaiXianlin",
                    value: "CaiXianlin"
                },
                {
                    text: "PetTrainer",
                    value: "PetTrainer"
                },
                {
                    text: "Petrainer 998DR",
                    value: "Petrainer998DR"
                }
            ],
            selectedDevice: "",
            newName: "",
            newRfId: 0,
            newModel: "CaiXianlin",
            devicesLoading: false,
            modal: {
                new: false,
                edit: false,
                editLoading: true
            },
            retrievedShocker: {
                id: "",
                rfId: 0,
                name: "",
                createdOn: "",
                model: 0
            }
        }
    },
    async beforeMount() {
        await this.loadShockers();
        this.emitter.on('refreshShockers', async () => {
            await this.loadShockers();
        });

        this.emitter.on('editShocker', async shockerId => {
            this.modal.editLoading = true;
            this.modal.edit = true;
            await this.loadDevices();
            await this.getShocker(shockerId);
        });
    },
    computed: {
        deviceList() {
            var arr = [];
            this.devices.forEach(it => {
                arr.push({
                    text: it.name,
                    value: it.id
                });
            });
            return arr;
        },
        validationDevice() {
            return this.selectedDevice !== "";
        },
        validationName() {
            return this.newName !== "" && this.newName.length >= 1 && this.newName.length <= 48;
        },
        validationRfId() {
            return this.newRfId >= 0 && this.newRfId <= 65535;
        },
        validationEditDevice() {
            return this.retrievedShocker.device !== "";
        },
        validationEditName() {
            return this.retrievedShocker.name !== "" && this.retrievedShocker.name.length >= 1 && this.retrievedShocker.name.length <= 48;
        },
        validationEditRfId() {
            return this.retrievedShocker.rfId >= 0 && this.retrievedShocker.rfId <= 65535;
        },
        validationEditModel() {
            var valid = false;
            this.modelOptions.forEach((text, value) => {
                if(value === this.retrievedShocker.model) valid = true;
            });
            return valid;
        }
    },
    methods: {
        async loadShockers() {
            const res = await apiCall.makeCall('GET', '1/shockers/own');
            if (res === undefined || res.status !== 200) {
                toastr.error("Error while retrieving own shockers");
                return;
            }

            this.ownShockers = res.data.data;
            this.ownShockers.forEach(device => {
                device.shockers.forEach(shocker => {
                    shocker.state = {
                        intensity: 25,
                        duration: 1
                    }
                    shocker.$live = {
                        dragging: false,
                        intensity: 0,
                        type: "vibrate"
                    }
                });
            });

            this.loadingShockers = false;
        },
        async loadDevices() {
            this.devicesLoading = true;
            const res = await apiCall.makeCall('GET', '1/devices');
            if (res === undefined || res.status !== 200) {
                toastr.error("Error while retrieving devices");
                return;
            }

            this.devices = res.data.data;
            this.devicesLoading = false;
        },
        async openNewShockerModal() {
            this.selectedDevice = "";
            this.newName = "New Shocker " + new Date().toISOString();
            this.newRfId = Math.floor(Math.random() * 65536);
            this.modal.new = true;
            await this.loadDevices();
        },
        async createNewShocker() {
            if(!this.validationDevice || !this.validationName || !this.validationRfId) return;

            const res = await apiCall.makeCall('POST', '1/shockers', {
                device: this.selectedDevice,
                name: this.newName,
                rfId: this.newRfId,
                model: this.newModel
            });
            if (res === undefined || res.status !== 201) {
                toastr.error("Error while creating new shocker");
                return;
            }
            
            this.modal.new = false;
            this.$swal('Success!', 'Successfully created new shocker', 'success');
            await this.loadShockers();
        },
        async getShocker(id) {
                const res = await apiCall.makeCall('GET', '1/shockers/' + id);
                if (res === undefined || res.status !== 200) {
                    toastr.error("Error while retrieving shocker details");
                    return;
                }

                this.retrievedShocker = res.data.data;
                this.modal.editLoading = false;
        },
        async applyEdits() {
            if(this.modal.editLoading) return;

            const res = await apiCall.makeCall('PATCH', '1/shockers/' + this.retrievedShocker.id, {
                name: this.retrievedShocker.name,
                device: this.retrievedShocker.device,
                rfId: this.retrievedShocker.rfId,
                model: this.retrievedShocker.model
            });

            if (res === undefined || res.status !== 200 && res.status !== 204) {
                toastr.error("Error while updating shocker");
                return;
            }

            this.$swal('Success!', `Successfully updated shocker [${this.retrievedShocker.id}]`, 'success');
            this.loadShockers();
            this.modal.edit = false;
        }
    }
}
</script>

<style scoped lang="scss">
.add-circle {
  position: fixed;
  right: 10px;
  bottom: 10px;
  width: 60px;
  height: 60px;


  background-color: #7ac142;
  border-radius: 50%;
  cursor: pointer;

  transition: background-color 0.2s;
  &:hover {
    background-color: #5e9634;
  }

  svg {
    height: 40px;
    width: 40px;

    position: relative;
    left: 52%;
    transform: translateX(-50%) translateY(-50%);
    top: 50%;

  }
}
</style>