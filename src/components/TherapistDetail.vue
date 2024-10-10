<template>
  <v-card>
    <v-card-title class="text-h5 d-flex justify-space-between">
      Therapist bearbeiten
      <v-btn icon @click="cancelChanges">
        <v-icon>mdi-close</v-icon>
      </v-btn>
    </v-card-title>
    <v-card-text v-if="therapistInput && therapistUser">
      <v-row>
        <v-col>
          <v-text-field label="Vorname" v-model="therapistInput.firstName" clearable></v-text-field>
        </v-col>
        <v-col>
          <v-text-field label="Nachname" v-model="therapistInput.lastName" clearable></v-text-field>
        </v-col>
        <v-col>
          <v-text-field label="Benutzername" v-model="therapistUser.username"></v-text-field>
        </v-col>
        <v-col cols="auto">
          <v-checkbox label="Aktiv" v-model="therapistInput.isActive"></v-checkbox>
        </v-col>
      </v-row>
      <v-row>
        <v-col>
          <div class="v-label">Aktiv Seit</div>
          <VueDatePicker
            v-model="therapistInput.activeSince"
            @change="handleDateSinceChange"
            text-input
            :format="formatDate"
            :format-locale="de"
            teleport-center
            select-text="Bestätigen"
            cancel-text="Abbrechen"
            :enable-time-picker="false"
          />
        </v-col>
        <v-col>
          <div class="v-label">Aktiv Bis</div>
          <VueDatePicker
            v-model="therapistInput.activeUntil"
            @change="handleDateUntilChange"
            :format="formatDate"
            :format-locale="de"
            teleport-center
            select-text="Bestätigen"
            cancel-text="Abbrechen"
            :enable-time-picker="false"
          />
        </v-col>
      </v-row>
      <v-row>
       <v-spacer></v-spacer>
      
        <!-- Tab Selector -->
      <v-tabs v-model="activeTab">
        <v-tab>Einzeltermine</v-tab>
        <v-tab>Serientermine</v-tab>
        <v-tab>Abwesenheiten</v-tab>
      </v-tabs>
      <v-spacer></v-spacer>
  
      </v-row>
    
      <v-tabs-items v-model="activeTab">
        <!-- Einzeltermine -->
        <v-tab-item v-if="activeTab === 0">
          <v-row>
            <v-data-table
              :headers="appointmentHeaders"
              :items="appointments"
              item-key="id"
              :loading="loadingAppointments"
              :loading-text="'Laden...'"
            >
        <template v-slot:item="{ item }">
          <tr @click="showSingleAppointmentDialog(item)" style="cursor: pointer;">
            <td>{{ item.patient.fullName }}</td>
            <td>{{ formatDate(item.date) }}</td>
            <td>{{ formatTime(item.startTime) }}</td>
            <td>{{ formatTime(item.endTime) }}</td>
            <td>{{ item.comment }}</td>
          </tr>
        </template>
            </v-data-table>
          </v-row>
        </v-tab-item>

        <!-- Serientermine -->
        <v-tab-item v-if="activeTab === 1">
          <v-row>
            <v-data-table
              :headers="appointmentSeriesHeaders"
              :items="appointmentSeries"
              item-key="id"
              :loading="loadingSeries"
              :loading-text="'Laden...'"
            >
            <template v-slot:item="{ item }">
              <tr @click="showAppointmentSeriesDialog(item)" style="cursor: pointer;">
                <td>{{ item.patient.fullName }}</td>
                <td>{{ item.weekday }}</td>
                <td>{{ formatTime(item.startTime) }}</td>
                <td>{{ formatTime(item.endTime) }}</td>
                <td>{{ formatDate(item.startDate) }}</td>
                <td>{{ formatDate(item.endDate) }}</td>
                <td>{{ item.weeklyFrequency }}</td>
                <td>{{ item.comment }}</td>
              </tr>
              </template>
            </v-data-table>
          </v-row>
        </v-tab-item>

        <!-- Abwesenheiten -->
        <v-tab-item v-if="activeTab === 2">
          <v-row>
            <v-data-table
              :headers="absenceHeaders"
              :items="absences"
              item-key="id"
              :loading="loadingAbsences"
              :loading-text="'Laden...'"
            > 
            <template v-slot:item="{ item }">
              <tr 
              @click="showAbsenceDialog(item)"
              :class="{'recurring-absence': !item.date}" 
              style="cursor: pointer;"
              >
                <td>{{ item.date ? formatDate(item.date) : item.date }}</td>
                <td>{{ item.weekday ? item.weekday + 's' : '' }}</td>
                <td>{{ formatTime(new Date(item.startTime)) }}</td>
                <td>{{ formatTime(new Date(item.endTime)) }}</td>
              </tr>
              </template>
            </v-data-table>
            <v-btn color="green" @click="openAddAbsenceDialog">
              <v-icon>mdi-plus</v-icon>
              Abwesenheit hinzufügen
            </v-btn>
          </v-row>
        </v-tab-item>
      </v-tabs-items>  
    </v-card-text>
    <v-card-actions>
      <v-btn color="grey" @click="cancelChanges">Abbrechen</v-btn>
      <v-spacer></v-spacer>
      <v-btn color="error" @click="deleteTherapist">Therapist löschen</v-btn>
      <v-spacer></v-spacer>
      <v-btn color="green" @click="saveChanges">Speichern</v-btn>
    </v-card-actions>
    <AbsenceDialog
      :therapistId="therapistId"
      :absence="absenceInput"
      v-model="isAbsenceDialogVisible"
      @save="changeAbsence"
      @delete="deleteAbsence"
      @cancel="isAbsenceDialogVisible = false"
    />
    <SingleAppointmentDialog
      v-if="selectedSingleAppointment"
      :appointment.sync="selectedSingleAppointment"
      :currentDay="new Date()"
      v-model="singleAppointmentDialog"
      @saveSingle="changeSingleAppointment"
      @saveSeries="changeSeriesAppointment"
      @deleteSingle="deleteSingleAppointment"
      @cancel="singleAppointmentDialog = false"
    />
    <AppointmentSeriesDialog
       v-if="selectedAppointmentSeries"
      :currentDay="new Date()"
      :appointment.sync="selectedAppointmentSeries"
      v-model="appointmentSeriesDialog"
      @saveSeries="changeSeriesAppointment"
      @cancel="appointmentSeriesDialog = false"
    />
</v-card>
<v-dialog v-model="deleteDialog" persistent max-width="290">
  <v-card>
    <v-card-title class="headline">{{ deleteDialogTitle }}</v-card-title>
    <v-card-text>{{ deleteDialogMessage }}</v-card-text>
    <v-card-actions>
      <v-spacer></v-spacer>
      <v-btn color="grey" text @click="deleteDialog = false">Abbrechen</v-btn>
      <v-btn color="error" text @click="confirmDelete">{{ deleteConfirmButtonText }}</v-btn>
    </v-card-actions>
  </v-card>
</v-dialog>

</template>

<script lang="ts">
import { defineComponent, ref, computed, onMounted } from 'vue';
import { useAppointmentStore } from '@/store/AppointmentStore';
import { useAppointmentSeriesStore } from '@/store/AppointmentSeriesStore';
import { useTherapistStore } from '@/store/TherapistStore';
import { de } from 'date-fns/locale';
import Therapist from '@/class/Therapist';
import Absence from '@/class/Absence';
import SingleAppointment from '@/class/SingleAppointment';
import AppointmentSeries from '@/class/AppointmentSeries';
import { useAbsenceStore } from '@/store/AbsenceStore';
import AbsenceDialog from './AbsenceDialog.vue';
import SingleAppointmentDialog from './SingleAppointmentDialog.vue';
import AppointmentSeriesDialog from './AppointmentSeriesDialog.vue';
import { formatDate, formatTime } from '@/class/Dateconversions';
import User from '@/class/User';

export default defineComponent({
  emits: ['save', 'cancel'],
  components: {
    AbsenceDialog,
    SingleAppointmentDialog,
    AppointmentSeriesDialog,
  },
  props: {
    therapistId: {
      type: Number,
      required: true,
    },
  },
  setup(props, { emit }) {
    const therapistInput = ref<Therapist | undefined>(undefined);
    const therapistUser = ref<User | undefined>(undefined);
    const appointments = ref<SingleAppointment[]>([]);
    const appointmentSeries = ref<AppointmentSeries[]>([]);
    const loadingAppointments = ref(true);
    const loadingSeries = ref(true);
    const loadingAbsences = ref(true);
    const absences = ref<Absence[]>([]);
    const activeTab = ref(0);
    const isAbsenceDialogVisible = ref(false);
    const editingAbsence = ref<Absence | null>(null);
    const absenceInput = ref<Absence | null>(null);
    const selectedSingleAppointment = ref<SingleAppointment | null>(null);
    const selectedAppointmentSeries = ref<AppointmentSeries | null>(null);
    const selectedAbsence = ref<Absence | null>(null);
    const singleAppointmentDialog = ref(false);
    const appointmentSeriesDialog = ref(false);
    const absenceDialog = ref(false);
    const deleteDialog = ref(false);
    const deleteDialogTitle = ref('Therapist Löschen?');
    const deleteDialogMessage = ref('');
    const deleteConfirmButtonText = ref('Löschen');

    const appointmentHeaders = ref([
      { title: 'Patient', value: 'patient.lastName' + ' ' + 'patient.firstName', sortable: true },
      { title: 'Datum', value: 'date', sortable: true },
      { title: 'Von', value: 'startTime', sortable: true },
      { title: 'Bis', value: 'endTime', sortable: true },
      { title: 'Kommentar', value: 'comment', sortable: true },
    ]);

    const appointmentSeriesHeaders = ref([
      { title: 'Patient', value: 'patient.lastName' + ' ' + 'patient.firstName', sortable: true },
      { title: 'Wochentag', value: 'weekday', sortable: true },
      { title: 'Von', value: 'startTime', sortable: true },
      { title: 'Bis', value: 'endTime', sortable: true },
      { title: 'Start Datum', value: 'startDate', sortable: true },
      { title: 'End Datum', value: 'endDate', sortable: true },
      { title: 'Interval', value: 'weeklyFrequency', sortable: true },
      { title: 'Kommentar', value: 'comment', sortable: true },
    ]);

    const absenceHeaders = ref([
      { title: 'Datum', value: 'date', sortable: true },
      { title: 'Wochentag', value: 'weekday', sortable: true },
      { title: 'Von', value: 'startTime', sortable: true },
      { title: 'Bis', value: 'endTime', sortable: true },
    ]);

    const therapistStore = useTherapistStore();
    const appointmentStore = useAppointmentStore();
    const appointmentSeriesStore = useAppointmentSeriesStore();
    const absenceStore = useAbsenceStore();

    const loadAppointments = async () => {
      await appointmentStore.loadAppointments({ therapistId: props.therapistId});
      loadingAppointments.value = true;
      appointments.value = await appointmentStore.getAppointmentsByTherapistId(props.therapistId);
      loadingAppointments.value = false;
    };

    const loadAppointmentSeries = async () => {
      loadingSeries.value = true;
      await appointmentSeriesStore.loadAppointmentSeries();
      appointmentSeries.value = await appointmentSeriesStore.getAppointmentSeriesByTherapistId(props.therapistId);
      loadingSeries.value = false;
    };

    const loadAbsences = async () => {
      loadingAbsences.value = true;
      await absenceStore.loadAbsences(props.therapistId);
      absences.value = await absenceStore.getAllAbsences;
      loadingAbsences.value = false;
    };

    onMounted(async () => {
      therapistInput.value = await therapistStore.getTherapistById(props.therapistId);
      therapistUser.value = await therapistStore.getTherapistUserById(props.therapistId);
      await loadAppointments();
      await loadAppointmentSeries();
      await loadAbsences();
    });

    const handleDateSinceChange = (date: Date) => {
      if (therapistInput.value) {
        therapistInput.value.activeSince = date;
      }
    };

    const handleDateUntilChange = (date: Date) => {
      if (therapistInput.value) {
        therapistInput.value.activeUntil = date;
      }
    }

    const cancelChanges = () => {
      emit('cancel');
    };

    const saveChanges = () => {
      if (therapistInput.value) {
        emit('save', { therapist: therapistInput.value });
      }
    };

    const deleteTherapist = async () => {
      try {
        // Überprüfe, ob der Therapeut noch Termine hat
        const appointments = await appointmentStore.getAppointmentsByTherapistId(therapistInput.value.id);

        if (appointments.length > 0) {
          // Falls Termine vorhanden sind, auf inaktiv setzen
          deleteDialogTitle.value = 'Therapist inaktiv setzen?';
          deleteDialogMessage.value = 'Der Therapeut hat noch bestehende Termine. Möchten Sie den Therapeuten auf inaktiv setzen?';
          deleteConfirmButtonText.value = 'Inaktiv setzen';
        } else {
          // Falls keine Termine vorhanden sind, löschen
          deleteDialogTitle.value = 'Therapist Löschen?';
          deleteDialogMessage.value = 'Möchten Sie diesen Therapeuten wirklich löschen? Diese Aktion kann nicht rückgängig gemacht werden.';
          deleteConfirmButtonText.value = 'Löschen';
        }

        deleteDialog.value = true;
      } catch (error) {
        console.error('Fehler beim Überprüfen der Termine:', error);
      }
    };

    const confirmDelete = async () => {
      try {
        const appointments = await appointmentStore.getAppointmentsByTherapistId(therapistInput.value.id);
        
        if (appointments.length > 0) {
          // Therapeuten auf inaktiv setzen, wenn Termine vorhanden sind
          therapistInput.value.isActive = false;
          console.log(therapistInput.value);
          
          await therapistStore.updateTherapist(therapistInput.value.id, therapistInput.value);
        } else {
          // Therapeuten löschen, wenn keine Termine vorhanden sind
          await therapistStore.deleteTherapist(therapistInput.value.id);
        }
      } catch (error) {
        console.error('Fehler beim Löschen/Inaktivsetzen des Therapeuten:', error);
        alert('Fehler beim Verarbeiten des Therapeuten.');
      }
      deleteDialog.value = false;
      emit('cancel');
    };

    const changeSingleAppointment = async (appointment: SingleAppointment) => {
       await appointmentStore.updateAppointment(appointment.id, appointment)
       loadAppointments();
    };

    const changeSeriesAppointment = (appointment: AppointmentSeries) => {
      appointmentSeriesStore.updateAppointmentSeries(appointment.id, appointment);
      loadAppointments();
    };


    const changeAbsence = async (absence: Absence) => {
      if(therapistInput.value === undefined) return;

      if (absence.id === 0) {
        absenceStore.addAbsence(therapistInput.value.id, absence);
        loadAbsences();
        closeAddAbsenceDialog();
      }
      else{
        absenceStore.updateAbsence(absence.id, absence);
        loadAbsences();
        closeAddAbsenceDialog();
      }
    };

    const deleteSingleAppointment = async (appointment: SingleAppointment) => {
      await appointmentStore.deleteAppointment(appointment.id);
      loadAppointments();
      closeAddAbsenceDialog();
    };

    const deleteSeriesAppointment = (id: number) => {
      appointmentSeriesStore.deleteAppointmentSeries(id);
      loadAppointments();
    };


    const openAddAbsenceDialog = () => {
      absenceInput.value = {
        id: 0,
        therapistId: props.therapistId,
        date: null,
        startTime: new Date(),
        endTime: new Date(),
        weekday: null,
      };
      isAbsenceDialogVisible.value = true;
    };

    const showSingleAppointmentDialog = (appointment: SingleAppointment) => {
      selectedSingleAppointment.value = appointment ; 
      singleAppointmentDialog.value = true;
    };

    const showAppointmentSeriesDialog = (appointmentSeries: AppointmentSeries) => {
      selectedAppointmentSeries.value = appointmentSeries;
      appointmentSeriesDialog.value = true;
    };

    const showAbsenceDialog = (absence: Absence) => {
      absenceInput.value = absence;
      isAbsenceDialogVisible.value = true;
    };

    const closeAddAbsenceDialog = () => {
      isAbsenceDialogVisible.value = false;
    };

    const openAbsenceDialog = () => {
      absenceInput.value = null;
      isAbsenceDialogVisible.value = true;
    };

    const editAbsence = (absence: Absence) => {
      absenceInput.value = absence;
      isAbsenceDialogVisible.value = true;
    };

    const deleteAbsence = async (absenceId: number) => {
      await absenceStore.deleteAbsence(props.therapistId, absenceId);
      isAbsenceDialogVisible.value = false;
    };

    return {
      therapistInput,
      de,
      appointments,
      appointmentSeries,
      absences,
      loadingAppointments,
      appointmentHeaders,
      appointmentSeriesHeaders,
      absenceHeaders,
      loadingSeries,
      loadingAbsences,
      activeTab,
      isAbsenceDialogVisible,
      editingAbsence,
      absenceInput,
      therapistUser,
      cancelChanges,
      saveChanges,
      deleteTherapist,
      deleteDialogTitle,
      deleteDialogMessage,
      deleteConfirmButtonText,
      deleteDialog,
      changeSingleAppointment,
      changeSeriesAppointment,
      deleteSingleAppointment,
      deleteSeriesAppointment,
      selectedSingleAppointment,
      selectedAppointmentSeries,
      singleAppointmentDialog,
      appointmentSeriesDialog,
      formatTime,
      formatDate,
      openAddAbsenceDialog,
      closeAddAbsenceDialog,
      handleDateSinceChange,
      handleDateUntilChange,
      openAbsenceDialog,
      editAbsence,
      deleteAbsence,
      showSingleAppointmentDialog,
      showAppointmentSeriesDialog,
      selectedAbsence,
      absenceDialog,
      showAbsenceDialog,
      changeAbsence,
      confirmDelete,
    };
  },
});
</script>

<style>

.tabs {
  padding-top: 40px;
}

.v-tab {
  padding-bottom: 10px;
  padding-top: 10px;
}

.v-data-table {
  padding-top: 15px;
}

.recurring-absence {
  background-color: #f0f0f0;
}
</style>
