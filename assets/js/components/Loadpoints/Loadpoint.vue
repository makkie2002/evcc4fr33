<template>
	<div class="loadpoint d-flex flex-column pt-4 pb-2 px-3 px-sm-4 mx-2 mx-sm-0">
		<div class="d-block d-sm-flex justify-content-between align-items-center mb-3">
			<div class="d-flex justify-content-between align-items-center mb-3 text-truncate">
				<h3 class="me-2 mb-0 text-truncate d-flex">
					<VehicleIcon
						v-if="chargerIcon"
						:name="chargerIcon"
						class="me-2 flex-shrink-0"
					/>
					<div class="text-truncate">
						{{ loadpointTitle }}
					</div>
				</h3>
				<LoadpointSettingsButton class="d-block d-sm-none" @click="openSettingsModal" />
			</div>
			<div class="mb-3 d-flex align-items-center">
				<Mode class="flex-grow-1" v-bind="modeProps" @updated="setTargetMode" />
				<LoadpointSettingsButton
					:id="id"
					class="d-none d-sm-block ms-2"
					@click="openSettingsModal"
				/>
			</div>
		</div>
		<LoadpointSettingsModal
			v-bind="settingsModal"
			@maxcurrent-updated="setMaxCurrent"
			@mincurrent-updated="setMinCurrent"
			@phasesconfigured-updated="setPhasesConfigured"
			@batteryboost-updated="setBatteryBoost"
		/>

		<div
			v-if="remoteDisabled"
			class="alert alert-warning my-4 py-2"
			:class="`${remoteDisabled === 'hard' ? 'alert-danger' : 'alert-warning'}`"
			role="alert"
		>
			{{
				$t(
					remoteDisabled === "hard"
						? "main.loadpoint.remoteDisabledHard"
						: "main.loadpoint.remoteDisabledSoft",
					{ source: remoteDisabledSource }
				)
			}}
		</div>

		<div class="details d-flex align-items-start mb-2">
			<div>
				<div class="d-flex align-items-center">
					<LabelAndValue
						:label="$t('main.loadpoint.power')"
						:value="chargePower"
						:valueFmt="fmtPower"
						class="mb-2 text-nowrap"
						align="start"
					/>
					<shopicon-regular-lightning
						class="text-evcc opacity-transiton"
						:class="`opacity-${showChargingIndicator ? '100' : '0'}`"
						size="m"
					></shopicon-regular-lightning>
				</div>
				<Phases
					v-bind="phasesProps"
					class="opacity-transiton"
					:class="`opacity-${showChargingIndicator ? '100' : '0'}`"
				/>
			</div>
			<LabelAndValue
				v-show="socBasedCharging"
				:label="$t('main.loadpoint.charged')"
				:value="fmtEnergy(chargedEnergy)"
				align="center"
			/>
			<LoadpointSessionInfo v-bind="sessionInfoProps" />
		</div>
		<hr class="divider" />
		<Vehicle
			class="flex-grow-1 d-flex flex-column justify-content-end"
			v-bind="vehicleProps"
			@limit-soc-updated="setLimitSoc"
			@limit-energy-updated="setLimitEnergy"
			@change-vehicle="changeVehicle"
			@remove-vehicle="removeVehicle"
			@open-loadpoint-settings="openSettingsModal"
		/>
	</div>
</template>

<script>
import "@h2d2/shopicons/es/regular/lightning";
import "@h2d2/shopicons/es/regular/adjust";
import api from "../../api.js";
import Mode from "./Mode.vue";
import Vehicle from "../Vehicles/Vehicle.vue";
import Phases from "./Phases.vue";
import LabelAndValue from "../Helper/LabelAndValue.vue";
import formatter, { POWER_UNIT } from "../../mixins/formatter.js";
import collector from "../../mixins/collector.js";
import SettingsButton from "./SettingsButton.vue";
import SettingsModal from "./SettingsModal.vue";
import VehicleIcon from "../VehicleIcon/index.js";
import SessionInfo from "./SessionInfo.vue";
import smartCostAvailable from "../../utils/smartCostAvailable.js";
import Modal from "bootstrap/js/dist/modal";

export default {
	name: "Loadpoint",
	components: {
		Mode,
		Vehicle,
		Phases,
		LabelAndValue,
		LoadpointSettingsButton: SettingsButton,
		LoadpointSettingsModal: SettingsModal,
		LoadpointSessionInfo: SessionInfo,
		VehicleIcon,
	},
	mixins: [formatter, collector],
	props: {
		id: Number,
		single: Boolean,

		// main
		title: String,
		mode: String,
		effectiveLimitSoc: Number,
		limitEnergy: Number,
		remoteDisabled: Boolean,
		remoteDisabledSource: String,
		chargeDuration: Number,
		charging: Boolean,
		batteryBoost: Boolean,
		batteryConfigured: Boolean,

		// session
		sessionEnergy: Number,
		sessionCo2PerKWh: Number,
		sessionPricePerKWh: Number,
		sessionPrice: Number,
		sessionSolarPercentage: Number,

		// charger
		chargerStatusReason: String,
		chargerFeatureIntegratedDevice: Boolean,
		chargerFeatureHeating: Boolean,
		chargerIcon: String,

		// vehicle
		connected: Boolean,
		// charging: Boolean,
		enabled: Boolean,
		vehicleDetectionActive: Boolean,
		vehicleRange: Number,
		vehicleSoc: Number,
		vehicleName: String,
		vehicleIcon: String,
		vehicleLimitSoc: Number,
		vehicles: Array,
		planActive: Boolean,
		planProjectedStart: String,
		planProjectedEnd: String,
		planOverrun: Number,
		planEnergy: Number,
		planTime: String,
		effectivePlanTime: String,
		effectivePlanSoc: Number,
		vehicleProviderLoggedIn: Boolean,
		vehicleProviderLoginPath: String,
		vehicleProviderLogoutPath: String,

		// details
		vehicleClimaterActive: Boolean,
		vehicleWelcomeActive: Boolean,
		chargePower: Number,
		chargedEnergy: Number,
		chargeRemainingDuration: Number,

		// other information
		phasesConfigured: Number,
		phasesActive: Number,
		chargerPhases1p3p: Boolean,
		chargerSinglePhase: Boolean,
		minCurrent: Number,
		maxCurrent: Number,
		chargeCurrent: Number,
		connectedDuration: Number,
		chargeCurrents: Array,
		chargeRemainingEnergy: Number,
		phaseAction: String,
		phaseRemaining: Number,
		pvRemaining: Number,
		pvAction: String,
		smartCostLimit: { type: Number, default: null },
		smartCostType: String,
		smartCostActive: Boolean,
		smartCostNextStart: String,
		tariffGrid: Number,
		tariffCo2: Number,
		currency: String,
		multipleLoadpoints: Boolean,
		gridConfigured: Boolean,
		pvConfigured: Boolean,
	},
	data() {
		return {
			tickerHandler: null,
			phaseRemainingInterpolated: this.phaseRemaining,
			pvRemainingInterpolated: this.pvRemaining,
			chargeDurationInterpolated: this.chargeDuration,
			chargeRemainingDurationInterpolated: this.chargeRemainingDuration,
		};
	},
	computed: {
		vehicle: function () {
			return this.vehicles?.find((v) => v.name === this.vehicleName);
		},
		vehicleTitle: function () {
			return this.vehicle?.title;
		},
		loadpointTitle: function () {
			return this.title || this.$t("main.loadpoint.fallbackName");
		},
		integratedDevice: function () {
			return this.chargerFeatureIntegratedDevice;
		},
		heating: function () {
			return this.chargerFeatureHeating;
		},
		phasesProps: function () {
			return this.collectProps(Phases);
		},
		modeProps: function () {
			return this.collectProps(Mode);
		},
		sessionInfoProps: function () {
			return this.collectProps(SessionInfo);
		},
		settingsModal: function () {
			return this.collectProps(SettingsModal);
		},
		vehicleProps: function () {
			return this.collectProps(Vehicle);
		},
		showChargingIndicator: function () {
			return this.charging && this.chargePower > 0;
		},
		vehicleKnown: function () {
			return !!this.vehicleName;
		},
		vehicleHasSoc: function () {
			return this.vehicleKnown && !this.vehicle?.features?.includes("Offline");
		},
		vehicleNotReachable: function () {
			// online vehicle that was not reachable at startup
			const features = this.vehicle?.features || [];
			return features.includes("Offline") && features.includes("Retryable");
		},
		planTimeUnreachable: function () {
			// 1 minute tolerance
			return this.planOverrun > 60;
		},
		socBasedCharging: function () {
			return this.vehicleHasSoc || this.vehicleSoc > 0;
		},
		socBasedPlanning: function () {
			return this.socBasedCharging && this.vehicle?.capacity > 0;
		},
		pvPossible: function () {
			return this.pvConfigured || this.gridConfigured;
		},
		hasSmartCost: function () {
			return smartCostAvailable(this.smartCostType);
		},
		batteryBoostAvailable: function () {
			return this.batteryConfigured && this.$hiddenFeatures();
		},
		batteryBoostActive: function () {
			return this.batteryBoost && this.charging && !["off", "now"].includes(this.mode);
		},
	},
	watch: {
		phaseRemaining() {
			this.phaseRemainingInterpolated = this.phaseRemaining;
		},
		pvRemaining() {
			this.pvRemainingInterpolated = this.pvRemaining;
		},
		chargeDuration() {
			this.chargeDurationInterpolated = this.chargeDuration;
		},
		chargeRemainingDuration() {
			this.chargeRemainingDurationInterpolated = this.chargeRemainingDuration;
		},
	},
	mounted() {
		this.tickerHandler = setInterval(this.tick, 1000);
	},
	unmounted() {
		clearInterval(this.tickerHandler);
	},
	methods: {
		tick() {
			if (this.phaseRemainingInterpolated > 0) {
				this.phaseRemainingInterpolated--;
			}
			if (this.pvRemainingInterpolated > 0) {
				this.pvRemainingInterpolated--;
			}
			if (this.chargeDurationInterpolated > 0 && this.charging) {
				this.chargeDurationInterpolated++;
			}
			if (this.chargeRemainingDurationInterpolated > 0 && this.charging) {
				this.chargeRemainingDurationInterpolated--;
			}
		},
		apiPath: function (func) {
			return "loadpoints/" + this.id + "/" + func;
		},
		setTargetMode: function (mode) {
			api.post(this.apiPath("mode") + "/" + mode);
		},
		setLimitSoc: function (soc) {
			api.post(this.apiPath("limitsoc") + "/" + soc);
		},
		setLimitEnergy: function (kWh) {
			api.post(this.apiPath("limitenergy") + "/" + kWh);
		},
		setMaxCurrent: function (maxCurrent) {
			api.post(this.apiPath("maxcurrent") + "/" + maxCurrent);
		},
		setMinCurrent: function (minCurrent) {
			api.post(this.apiPath("mincurrent") + "/" + minCurrent);
		},
		setPhasesConfigured: function (phases) {
			api.post(this.apiPath("phases") + "/" + phases);
		},
		changeVehicle(name) {
			api.post(this.apiPath("vehicle") + `/${name}`);
		},
		removeVehicle() {
			api.delete(this.apiPath("vehicle"));
		},
		setBatteryBoost: function (batteryBoost) {
			api.post(this.apiPath("batteryboost") + `/${batteryBoost ? "1" : "0"}`);
		},
		fmtPower(value) {
			return this.fmtW(value, POWER_UNIT.AUTO);
		},
		fmtEnergy(value) {
			return this.fmtWh(value, POWER_UNIT.AUTO);
		},
		openSettingsModal() {
			const modal = Modal.getOrCreateInstance(
				document.getElementById(`loadpointSettingsModal_${this.id}`)
			);
			modal.show();
		},
	},
};
</script>

<style scoped>
.loadpoint {
	border-radius: 2rem;
	color: var(--evcc-default-text);
	background: var(--evcc-box);
}

.details > div {
	flex-grow: 1;
	flex-basis: 0;
}
.details > div:nth-child(2) {
	text-align: center;
}
.details > div:nth-child(3) {
	text-align: right;
}
.opacity-transiton {
	transition: opacity var(--evcc-transition-slow) ease-in;
}
.divider {
	border: none;
	border-bottom-width: 1px;
	border-bottom-style: solid;
	border-bottom-color: var(--evcc-gray);
	background: none;
	opacity: 0.5;
	margin: 0 -1rem;
}
/* breakpoint sm */
@media (min-width: 576px) {
	.divider {
		margin: 0 -1.5rem;
	}
}
</style>
