<template>
  <v-dialog persistent v-model="dialogIsOpen" width="800">
    <v-card>
      <v-card-title class="text-h5 grey lighten-2">
        <v-row>
          <v-col>
            Einzeltermin - {{ appointment.therapist.firstName }}
          </v-col>
          <v-col cols="auto" v-if="appointment.createdBySeriesAppointment">
            <v-btn icon @click="handleSeriesButtonClick">
              <v-icon>mdi-calendar-multiple</v-icon>
            </v-btn>
          </v-col>
        </v-row>
      </v-card-title>

      <v-card-text class="pt-4">
        <v-form>
          <!-- Patientensuche -->
          <v-autocomplete
            v-model="selectedPatientId"
            :items="patientsOptions"
            item-title="name"
            item-value="id"
            label="Patient suchen"
            :placeholder="singleAppointment.patient?.fullName || 'Patient suchen'"
            @update:search-input="handleSearchInput"
          >
            <template v-slot:item="data">
              <v-list-item v-bind="data.props">
              </v-list-item>
            </template>
          </v-autocomplete>

          <!-- Zeitpicker für Einzeltermin -->
          <v-row>
            <v-col>
              <div class="v-label">Von</div>
              <VueDatePicker
                v-model="singleAppointment.startTime"
                :format-locale="de"
                :format="formatTime"
                minutes-increment="10"
                teleport-center
                select-text="Bestätigen"
                cancel-text="Abbrechen"
              />
            </v-col>
            <v-col>
              <div class="v-label">Bis</div>
              <VueDatePicker
                v-model="singleAppointment.endTime"
                :format-locale="de"
                :format="formatTime"
                minutes-increment="10"
                teleport-center
                select-text="Bestätigen"
                cancel-text="Abbrechen"
              />
            </v-col>
          </v-row>

          <!-- Behandlungsmöglichkeiten -->
          <v-row>
            <v-col>
              <v-checkbox
                v-model="singleAppointment.isHotair"
                label="Heißluft"
              ></v-checkbox>
            </v-col>
            <v-col>
              <v-checkbox
                v-model="singleAppointment.isElectric"
                label="Elektro"
              ></v-checkbox>
            </v-col>
            <v-col>
              <v-checkbox
                v-model="singleAppointment.isUltrasonic"
                label="Ultraschall"
              ></v-checkbox>
            </v-col>
          </v-row>

          <!-- Kommentar -->
          <v-text-field
            v-model="singleAppointment.comment"
            label="Kommentar"
          ></v-text-field>

          <!-- Serientermin Felder -->
          <div v-if="isSeries">
            <v-form>
              <v-row>
                <v-col>
                  <div class="v-label">Datum Von</div>
                  <VueDatePicker
                    v-model="seriesAppointment.startDate"
                    :format-locale="de"
                    :format="formatDate"
                    :enable-time-picker="false"
                    teleport-center
                    select-text="Bestätigen"
                    cancel-text="Abbrechen"
                  />
                </v-col>
                <v-col>
                  <div class="v-label">Datum Bis</div>
                  <VueDatePicker
                    v-model="seriesAppointment.endDate"
                    :format-locale="de"
                    :format="formatDate"
                    :enable-time-picker="false"
                    teleport-center
                    select-text="Bestätigen"
                    cancel-text="Abbrechen"
                  />
                </v-col>
              </v-row>

              <v-text-field
                v-model="seriesAppointment.weeklyFrequency"
                label="Intervall (Woche)"
                type="number"
              />
            </v-form>
          </div>
        </v-form>
      </v-card-text>

      <v-card-actions>
        <v-btn color="grey" @click="closeDialog">Abbrechen</v-btn>
        <v-spacer></v-spacer>
        <v-btn v-if="appointment.id" color="error" @click="deleteAppointment">Termin Löschen</v-btn>
        <v-spacer></v-spacer>
        <v-btn
          :disabled="!isSeries ? !isValid : !isSeriesValid"
          color="green"
          @click="saveAppointment"
        >
          Speichern
        </v-btn>
      </v-card-actions>
    </v-card>
        <AppointmentSeriesDialog
          :currentDay="currentDay"
          :appointment="seriesAppointment"
          v-model="seriesAppointmentDialogOpen"
          @saveSeries="handleSeriesSave"
          @cancel="seriesAppointmentDialogOpen = false"
        />
  </v-dialog>
  
</template>

<script lang="ts">
import { defineComponent, ref, computed, watch, onMounted } from 'vue';
import SingleAppointment from '@/class/SingleAppointment';
import AppointmentSeries from '@/class/AppointmentSeries';
import { usePatientStore } from '@/store/PatientStore';
import { Weekday } from '@/class/Enums';
import { de } from 'date-fns/locale';
import Patient from '@/class/Patient';
import AppointmentSeriesDialog from './AppointmentSeriesDialog.vue';
import { useAppointmentSeriesStore } from '@/store/AppointmentSeriesStore';
import { formatDate, formatTime } from '@/class/Dateconversions';

export default defineComponent({
  components: {
    AppointmentSeriesDialog
  },
  props: {
    currentDay: {
      type: Date,
      required: true,
    },
    appointment: {
      type: Object as () => SingleAppointment,
      required: true,
    },
  },
  emits: ['saveSingle', 'saveSeries', 'deleteSingle', 'cancel'],
  setup(props, { emit }) {
    const dialogIsOpen = ref(false);
    const isSeries = ref(false);
    const createPatientDialogOpen = ref(false);
    const seriesAppointmentDialogOpen = ref(false);
    const appointmentSeriesStore = useAppointmentSeriesStore();
    const patientStore = usePatientStore();
    const patients = ref<Patient[]>([]);
    const patientsOptions = ref<{ id: number, name: string }[]>([]);

    const singleAppointment = ref<SingleAppointment>(props.appointment);

    const seriesAppointment = ref<AppointmentSeries>(new AppointmentSeries(
      props.appointment.id,
      props.appointment.therapist,
      props.appointment.therapistId,
      props.appointment.patient,
      props.appointment.patientId,
      props.appointment.startTime,
      props.appointment.endTime,
      new Date(),
      new Date(),
      props.appointment.comment,
      Weekday.MONDAY,
      1,
      [],
      []
    ));

    onMounted(() => {
      loadPatients();
    });

    watch(() => props.appointment, (newAppointment) => {
      singleAppointment.value = newAppointment;
      selectedPatientId.value = newAppointment.patient?.id || null;
    });

    const loadPatients = async () => {
      await patientStore.loadPatients();
      await appointmentSeriesStore.loadAppointmentSeries();
      patients.value = patientStore.getAllPatients;
    };

    const selectedPatientId = ref<number | null>(singleAppointment.value.patient?.id || null);

    const isValid = computed(() =>
      singleAppointment.value.patient &&
      singleAppointment.value.startTime &&
      singleAppointment.value.endTime
    );

    const isSeriesValid = computed(() =>
      seriesAppointment.value.startDate &&
      seriesAppointment.value.endDate &&
      seriesAppointment.value.weeklyFrequency
    );

    watch(() => patientStore.getAllPatients, (newPatients) => {
      patients.value = newPatients;
      patientsOptions.value = newPatients.map(patient => ({
        id: patient.id,
        name: patient.firstName + ' ' + patient.lastName
      }));
    });

    watch(selectedPatientId, (newId) => {
      if (newId) {
        const patient = patients.value.find(p => p.id === newId);
        if (patient) {
          singleAppointment.value.patient = patient;
          singleAppointment.value.patientId = patient.id;
        }
      }
    });

    const saveAppointment = () => {
      if (isSeries.value) {
        if (!seriesAppointment.value.startDate || !seriesAppointment.value.endDate || !seriesAppointment.value.weeklyFrequency) {
          alert('Bitte alle Felder für den Serientermin ausfüllen.');
          return;
        }
        if (!selectedPatientId.value) {
          alert('Bitte einen gültigen Patienten auswählen.');
          return;
        }
        saveSeriesAppointment();
      } else {
        if (!singleAppointment.value.patient || !singleAppointment.value.startTime || !singleAppointment.value.endTime) {
          alert('Bitte alle Felder für den Einzeltermin ausfüllen.');
          return;
        }
        saveSingleAppointment();
      }
    };

    const saveSingleAppointment = () => {
      emit('saveSingle', singleAppointment.value);
      closeDialog();
    };

    const deleteAppointment = () => {
      emit('deleteSingle', singleAppointment.value);
      closeDialog();
    };

    const saveSeriesAppointment = () => {
      // Übertrage die Werte von singleAppointment auf seriesAppointment
      seriesAppointment.value.patient = singleAppointment.value.patient;
      seriesAppointment.value.patientId = singleAppointment.value.patientId;
      seriesAppointment.value.startTime = singleAppointment.value.startTime;
      seriesAppointment.value.endTime = singleAppointment.value.endTime;
      seriesAppointment.value.comment = singleAppointment.value.comment;

      emit('saveSeries', seriesAppointment.value);
      closeDialog();
    };

    const openCreatePatientDialog = () => {
      createPatientDialogOpen.value = true;
    };

    const closeDialog = () => {
      emit('cancel');
    };

    const handleSeriesButtonClick = () => {
      const appointmentSeries = appointmentSeriesStore.getAppointmentSeriesById(singleAppointment.value.appointmentSeriesId);
      if (appointmentSeries) {
        console.log(appointmentSeries);
        seriesAppointment.value = appointmentSeries;
        seriesAppointmentDialogOpen.value = true;
      }
    };

    const handleSeriesSave = () => {
      saveAppointment();
      seriesAppointmentDialogOpen.value = false;
    };

    const handleSeriesCancel = () => {
      seriesAppointmentDialogOpen.value = false;
    };

    const handleSearchInput = (search: string) => {
      // Optionale Filterlogik basierend auf der Suchanfrage
    };

    return {
      dialogIsOpen,
      isSeries,
      singleAppointment,
      seriesAppointment,
      patientsOptions,
      selectedPatientId,
      isValid,
      isSeriesValid,
      openCreatePatientDialog,
      seriesAppointmentDialogOpen,
      closeDialog,
      saveAppointment,
      deleteAppointment,
      handleSearchInput,
      handleSeriesButtonClick,
      handleSeriesSave,
      handleSeriesCancel,
      createPatientDialogOpen,
      de,
      formatTime,
      formatDate,
    };
  },
});
</script>
