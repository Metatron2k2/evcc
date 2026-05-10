<template>
	<div class="app app--bottomtabs">
		<PinLockOverlay v-if="showUiLock" @unlocked="onUiLockUnlocked" />
		<router-view
			v-if="showRoutes"
			:notifications="notifications"
			:offline="offline"
		></router-view>

		<BottomTabBar v-bind="bottomTabBarProps" />

		<GlobalSettingsModal v-bind="globalSettingsProps" />
		<AboutModal v-bind="aboutModalProps" />
		<HelpModal />
		<PasswordModal />
		<LoginModal v-bind="loginModalProps" />
		<OfflineIndicator v-bind="offlineIndicatorProps" />
	</div>
</template>

<script lang="ts">
import store from "../store";
import BottomTabBar from "../components/BottomTabs/Bar.vue";
import GlobalSettingsModal from "../components/GlobalSettings/GlobalSettingsModal.vue";
import OfflineIndicator from "../components/Footer/OfflineIndicator.vue";
import PasswordModal from "../components/Auth/PasswordModal.vue";
import PinLockOverlay from "../components/Auth/PinLockOverlay.vue";
import LoginModal from "../components/Auth/LoginModal.vue";
import AboutModal from "../components/AboutModal.vue";
import api from "../api";
import HelpModal from "../components/HelpModal.vue";
import collector from "../mixins/collector";
import { defineComponent } from "vue";

// assume offline if not data received for 5 minutes
let lastDataReceived = new Date();
const maxDataAge = 60 * 1000 * 5;
setInterval(() => {
	if (new Date().getTime() - lastDataReceived.getTime() > maxDataAge) {
		console.log("no data received, assume we are offline");
		window.app.setOffline();
	}
}, 1000);

export default defineComponent({
	name: "App",
	components: {
		AboutModal,
		BottomTabBar,
		GlobalSettingsModal,
		HelpModal,
		PasswordModal,
		PinLockOverlay,
		LoginModal,
		OfflineIndicator,
	},
	mixins: [collector],
	props: {
		notifications: Array,
		offline: Boolean,
	},
	data: () => {
		return {
			reconnectTimeout: null as number | null,
			ws: null as WebSocket | null,
			authNotConfigured: false,
			showUiLock: false,
			idleTimer: null as number | null,
			refreshTimer: null as number | null,
			idleActivityHandler: null as (() => void) | null,
			uilockTimeoutMs: 0,
			lastActivityTs: 0,
		};
	},
	head() {
		return { title: "...", titleTemplate: "%s | evcc" };
	},
	computed: {
		version() {
			return store.state.version;
		},
		showRoutes() {
			return this.state.startupCompleted;
		},
		state() {
			const { state, uiLoadpoints } = store;
			return { ...state, uiLoadpoints: uiLoadpoints.value };
		},
		globalSettingsProps() {
			return this.collectProps(GlobalSettingsModal, this.state);
		},
		offlineIndicatorProps() {
			return this.collectProps(OfflineIndicator, this.state);
		},
		loginModalProps() {
			return this.collectProps(LoginModal, this.state);
		},
		aboutModalProps() {
			return {
				installed: window.evcc.version,
				commit: window.evcc.commit,
				...this.collectProps(AboutModal, this.state),
			};
		},
		bottomTabBarProps() {
			return {
				installed: window.evcc.version,
				commit: window.evcc.commit,
				...this.collectProps(BottomTabBar, this.state),
			};
		},
	},
	watch: {
		version(now, prev) {
			if (!!prev && !!now) {
				console.log("new version detected. reloading browser", { now, prev });
				this.reload();
			}
		},
		offline(offline) {
			store.offline(offline);
			if (offline) {
				this.reconnect();
			}
		},
	},
	mounted() {
		void this.bootstrapUiLock();
		document.addEventListener("visibilitychange", this.pageVisibilityChanged, false);
		window.addEventListener("pageshow", this.pageShowHandler);
	},
	unmounted() {
		this.disconnect();
		this.clearReconnectTimeout();
		this.stopIdleMonitor();
		document.removeEventListener("visibilitychange", this.pageVisibilityChanged, false);
		window.removeEventListener("pageshow", this.pageShowHandler);
	},
	methods: {
		async bootstrapUiLock() {
			try {
				const res = await api.get("auth/uilock/status", {
					validateStatus: (s) => s >= 200 && s < 500,
				});
				if (res.status >= 500) {
					throw new Error(`status endpoint returned ${res.status}`);
				}
				const data = res.data as {
					appliesToClient?: boolean;
					unlocked?: boolean;
					timeout?: number;
				};
				if (data.appliesToClient && !data.unlocked) {
					store.update({ startupCompleted: true });
					this.showUiLock = true;
				} else if (data.appliesToClient && data.unlocked && data.timeout) {
					this.startIdleMonitor(data.timeout);
				}
			} catch (e) {
				console.warn("uilock status", e);
				if (store.state.uilock?.enabled && store.state.uilock?.pinConfigured) {
					store.update({ startupCompleted: true });
					this.showUiLock = true;
				}
			}
			this.connect();
		},
		async onUiLockUnlocked() {
			this.showUiLock = false;
			try {
				const res = await api.get("auth/uilock/status", {
					validateStatus: (s) => s >= 200 && s < 500,
				});
				const timeout = (res.data as { timeout?: number }).timeout;
				if (timeout) this.startIdleMonitor(timeout);
			} catch {
				// fall back to store value if available
				const timeout = store.state.uilock?.timeout as number | undefined;
				if (timeout) this.startIdleMonitor(timeout);
			}
		},
		startIdleMonitor(timeoutSec: number) {
			this.stopIdleMonitor();
			if (timeoutSec <= 0) return;

			this.uilockTimeoutMs = timeoutSec * 1000;
			this.lastActivityTs = Date.now();

			this.idleActivityHandler = () => {
				const now = Date.now();
				if (now - this.lastActivityTs < 1000) return;
				this.lastActivityTs = now;
				this.resetIdleTimer();
			};
			const events = ["mousedown", "keydown", "touchstart", "scroll"];
			events.forEach((e) =>
				document.addEventListener(e, this.idleActivityHandler!, { passive: true })
			);

			this.resetIdleTimer();

			const refreshMs = Math.min(60_000, Math.max(30_000, this.uilockTimeoutMs / 3));
			this.refreshTimer = window.setInterval(() => {
				if (!this.showUiLock && Date.now() - this.lastActivityTs < this.uilockTimeoutMs) {
					api.get("auth/uilock/status", { validateStatus: (s: number) => s < 500 });
				}
			}, refreshMs);
		},
		resetIdleTimer() {
			if (this.idleTimer) window.clearTimeout(this.idleTimer);
			this.idleTimer = window.setTimeout(() => {
				this.showUiLock = true;
				this.stopIdleMonitor();
			}, this.uilockTimeoutMs);
		},
		stopIdleMonitor() {
			if (this.idleTimer) {
				window.clearTimeout(this.idleTimer);
				this.idleTimer = null;
			}
			if (this.refreshTimer) {
				window.clearInterval(this.refreshTimer);
				this.refreshTimer = null;
			}
			if (this.idleActivityHandler) {
				const events = ["mousedown", "keydown", "touchstart", "scroll"];
				events.forEach((e) => document.removeEventListener(e, this.idleActivityHandler!));
				this.idleActivityHandler = null;
			}
		},
		clearReconnectTimeout() {
			if (this.reconnectTimeout) {
				window.clearTimeout(this.reconnectTimeout);
			}
		},
		pageShowHandler(event: PageTransitionEvent) {
			if (event.persisted) {
				this.clearReconnectTimeout();
				this.disconnect();
				this.connect();
			}
		},
		pageVisibilityChanged() {
			// disconnect in any case to ensure fresh connection
			this.clearReconnectTimeout();
			this.disconnect();
			if (!document.hidden) {
				this.connect();
			}
		},
		reconnect() {
			this.clearReconnectTimeout();
			this.reconnectTimeout = window.setTimeout(() => {
				this.disconnect();
				this.connect();
			}, 2500);
		},
		disconnect() {
			if (this.ws) {
				this.ws.onerror = null;
				this.ws.onopen = null;
				this.ws.onclose = null;
				this.ws.onmessage = null;
				this.ws.close();
				this.ws = null;
			}
		},
		connect() {
			console.log("websocket connect");
			const supportsWebSockets = "WebSocket" in window;
			if (!supportsWebSockets) {
				window.app.raise({
					message: "Web sockets not supported. Please upgrade your browser.",
				});
				return;
			}

			if (this.ws) {
				console.log("websocket already connected");
				return;
			}

			const loc = new URL("ws", window.location.href);
			loc.protocol = window.location.protocol === "https:" ? "wss:" : "ws:";

			this.ws = new WebSocket(loc.href);
			this.ws.onerror = () => {
				console.log({ message: "Websocket error. Trying to reconnect." });
				this.ws?.close();
			};
			this.ws.onopen = () => {
				console.log("websocket connected");
				window.app.setOnline();
			};
			this.ws.onclose = () => {
				window.app.setOffline();
				this.reconnect();
			};
			this.ws.onmessage = (evt) => {
				try {
					const msg = JSON.parse(evt.data);
					if (msg.startupCompleted) {
						store.reset();
					}
					store.update(msg);
					lastDataReceived = new Date();
				} catch (error) {
					const e = error as Error;
					window.app.raise({
						message: `Failed to parse web socket data: ${e.message} [${evt.data}]`,
					});
				}
			};
		},
		reload() {
			window.location.reload();
		},
	},
});
</script>
<style scoped>
.app {
	min-height: 100vh;
	min-height: 100dvh;
}
.app--bottomtabs {
	--bottom-space: calc(var(--tab-bar-height) + 1.5rem);
}
</style>
