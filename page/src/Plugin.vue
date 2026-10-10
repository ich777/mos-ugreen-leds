<template>
  <div>
    <h2 class="mb-4">{{ $t('plugin_ugreen_leds.title') }}</h2>

    <v-skeleton-loader v-if="loading" :loading="true" type="card" />

    <div v-else style="margin-bottom: 80px">
      <!-- Status Card -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-led-strip-variant</v-icon>
          <span>{{ $t('plugin_ugreen_leds.status') }}</span>
        </v-card-title>
        <v-card-text>
          <v-alert
            v-if="!status.module_loaded"
            type="warning" variant="tonal" density="compact" class="mb-3"
            :text="$t('plugin_ugreen_leds.module_not_loaded')"
          />
          <v-row dense>
            <v-col cols="12" md="4">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_ugreen_leds.model') }}</strong></div>
              <div class="text-body-2">{{ detect.model || '-' }}</div>
            </v-col>
            <v-col cols="6" md="4">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_ugreen_leds.driver') }}</strong></div>
              <div class="text-body-2">
                {{ status.module_loaded ? $t('plugin_ugreen_leds.loaded') : $t('plugin_ugreen_leds.not_loaded') }}
              </div>
            </v-col>
            <v-col cols="6" md="4">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_ugreen_leds.monitor') }}</strong></div>
              <div class="text-body-2">
                {{ status.running ? $t('plugin_ugreen_leds.running') : $t('plugin_ugreen_leds.not_running') }}
              </div>
            </v-col>
          </v-row>

          <div v-if="status.leds.length" class="d-flex flex-wrap ga-3 mt-4">
            <div v-for="led in status.leds" :key="led.name" class="d-flex flex-column align-center" style="min-width: 56px">
              <div :style="ledStyle(led)" />
              <div class="text-caption mt-1">{{ ledLabel(led.name) }}</div>
              <div class="text-caption text-medium-emphasis">{{ led.state || '-' }}</div>
            </div>
          </div>

          <div class="d-flex flex-wrap ga-2 mt-4">
            <v-btn size="small" variant="tonal" color="primary" :loading="busy === 'start'" @click="runAction('start')">
              <v-icon start>mdi-play</v-icon>{{ $t('plugin_ugreen_leds.start') }}
            </v-btn>
            <v-btn size="small" variant="tonal" color="error" :loading="busy === 'stop'" @click="runAction('stop')">
              <v-icon start>mdi-stop</v-icon>{{ $t('plugin_ugreen_leds.stop') }}
            </v-btn>
            <v-btn size="small" variant="tonal" color="secondary" :loading="busy === 'off'" @click="runAction('off')">
              <v-icon start>mdi-lightbulb-off</v-icon>{{ $t('plugin_ugreen_leds.all_off') }}
            </v-btn>
            <v-btn size="small" variant="tonal" color="secondary" :loading="busy === 'log'" @click="showLog">
              <v-icon start>mdi-text-box-outline</v-icon>{{ $t('plugin_ugreen_leds.show_log') }}
            </v-btn>
          </div>
        </v-card-text>
      </v-card>

      <!-- General -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-cog</v-icon>
          <span>{{ $t('plugin_ugreen_leds.settings') }}</span>
        </v-card-title>
        <v-card-text>
          <v-switch
            v-model="settings.enabled"
            :label="$t('plugin_ugreen_leds.enabled')"
            :hint="$t('plugin_ugreen_leds.enabled_hint')"
            persistent-hint inset color="green"
          />
        </v-card-text>
      </v-card>

      <!-- Power LED -->
      <v-card class="mb-4 pa-0" :disabled="!settings.enabled">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-power</v-icon>
          <span>{{ $t('plugin_ugreen_leds.power_led') }}</span>
        </v-card-title>
        <v-card-text>
          <v-switch v-model="settings.power.enabled" :label="$t('plugin_ugreen_leds.led_enabled')" inset color="green" hide-details />
          <v-row v-if="settings.power.enabled" class="mt-2">
            <v-col cols="12" md="4">
              <ColorField v-model="settings.power.color" :label="$t('plugin_ugreen_leds.color')" />
            </v-col>
            <v-col cols="12" md="4">
              <v-select
                v-model="settings.power.mode"
                :items="powerModes"
                :label="$t('plugin_ugreen_leds.mode')"
                density="comfortable" hide-details
              />
            </v-col>
            <v-col cols="12" md="4">
              <v-slider
                v-model="settings.power.brightness"
                :label="$t('plugin_ugreen_leds.brightness')"
                :min="1" :max="255" :step="1" thumb-label hide-details
              />
            </v-col>
            <template v-if="settings.power.mode !== 'on'">
              <v-col cols="6" md="4">
                <v-text-field
                  v-model.number="settings.power.on_ms" type="number" :min="100" :max="32767"
                  :label="$t('plugin_ugreen_leds.on_ms')" density="comfortable" hide-details
                />
              </v-col>
              <v-col cols="6" md="4">
                <v-text-field
                  v-model.number="settings.power.off_ms" type="number" :min="100" :max="32767"
                  :label="$t('plugin_ugreen_leds.off_ms')" density="comfortable" hide-details
                />
              </v-col>
            </template>
          </v-row>
        </v-card-text>
      </v-card>

      <!-- Network LED -->
      <v-card class="mb-4 pa-0" :disabled="!settings.enabled">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-ethernet</v-icon>
          <span>{{ $t('plugin_ugreen_leds.network_led') }}</span>
        </v-card-title>
        <v-card-text>
          <v-switch v-model="settings.network.enabled" :label="$t('plugin_ugreen_leds.led_enabled')" inset color="green" hide-details />
          <template v-if="settings.network.enabled">
            <v-row class="mt-2">
              <v-col cols="12" md="4">
                <v-select
                  v-model="settings.network.interface"
                  :items="interfaceItems"
                  :label="$t('plugin_ugreen_leds.interface')"
                  density="comfortable" hide-details
                />
              </v-col>
              <v-col cols="12" md="4">
                <v-select
                  v-model="settings.network.color_mode"
                  :items="colorModes"
                  :label="$t('plugin_ugreen_leds.color_mode')"
                  density="comfortable" hide-details
                />
              </v-col>
              <v-col cols="12" md="4">
                <v-slider
                  v-model="settings.network.brightness"
                  :label="$t('plugin_ugreen_leds.brightness')"
                  :min="1" :max="255" :step="1" thumb-label hide-details
                />
              </v-col>
            </v-row>

            <v-row v-if="settings.network.color_mode === 'static'">
              <v-col cols="12" md="4">
                <ColorField v-model="settings.network.color" :label="$t('plugin_ugreen_leds.color')" />
              </v-col>
            </v-row>
            <v-row v-else-if="settings.network.color_mode === 'link_speed'">
              <v-col v-for="speed in linkSpeeds" :key="speed" cols="6" md="2">
                <ColorField v-model="settings.network['color_' + speed]" :label="speedLabel(speed)" />
              </v-col>
              <v-col cols="6" md="2">
                <ColorField v-model="settings.network.color" :label="$t('plugin_ugreen_leds.color_other_speed')" />
              </v-col>
            </v-row>
            <v-row v-else>
              <v-col cols="6" md="3">
                <ColorField v-model="settings.network.dynamic_color_low" :label="$t('plugin_ugreen_leds.color_low')" />
              </v-col>
              <v-col cols="6" md="3">
                <ColorField v-model="settings.network.dynamic_color_high" :label="$t('plugin_ugreen_leds.color_high')" />
              </v-col>
              <v-col cols="6" md="3">
                <v-text-field
                  v-model.number="settings.network.dynamic_speed_low" type="number"
                  :label="$t('plugin_ugreen_leds.speed_low')" suffix="Mbit/s" density="comfortable" hide-details
                />
              </v-col>
              <v-col cols="6" md="3">
                <v-text-field
                  v-model.number="settings.network.dynamic_speed_high" type="number"
                  :label="$t('plugin_ugreen_leds.speed_high')" suffix="Mbit/s" density="comfortable" hide-details
                />
              </v-col>
            </v-row>

            <v-row>
              <v-col cols="6" md="3">
                <v-switch v-model="settings.network.blink_tx" :label="$t('plugin_ugreen_leds.blink_tx')" inset color="green" hide-details />
              </v-col>
              <v-col cols="6" md="3">
                <v-switch v-model="settings.network.blink_rx" :label="$t('plugin_ugreen_leds.blink_rx')" inset color="green" hide-details />
              </v-col>
            </v-row>

            <v-row>
              <v-col cols="12" md="6">
                <v-switch
                  v-model="settings.network.check_gateway"
                  :label="$t('plugin_ugreen_leds.check_gateway')"
                  inset color="green" hide-details
                />
              </v-col>
              <v-col v-if="settings.network.check_gateway" cols="12" md="4">
                <ColorField v-model="settings.network.color_gateway_unreachable" :label="$t('plugin_ugreen_leds.color_gateway_unreachable')" />
              </v-col>
            </v-row>
          </template>
        </v-card-text>
      </v-card>

      <!-- Disk LEDs -->
      <v-card class="mb-4 pa-0" :disabled="!settings.enabled">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-harddisk</v-icon>
          <span>{{ $t('plugin_ugreen_leds.disk_leds') }}</span>
        </v-card-title>
        <v-card-text>
          <v-switch v-model="settings.disks.enabled" :label="$t('plugin_ugreen_leds.led_enabled')" inset color="green" hide-details />
          <template v-if="settings.disks.enabled">
            <v-row class="mt-2">
              <v-col cols="12" md="4">
                <v-select
                  v-model="settings.disks.mapping"
                  :items="mappingMethods"
                  :label="$t('plugin_ugreen_leds.mapping')"
                  :hint="$t('plugin_ugreen_leds.mapping_hint')"
                  persistent-hint density="comfortable"
                />
              </v-col>
              <v-col cols="12" md="4">
                <v-slider
                  v-model="settings.disks.brightness"
                  :label="$t('plugin_ugreen_leds.brightness')"
                  :min="1" :max="255" :step="1" thumb-label hide-details
                />
              </v-col>
              <v-col cols="12" md="4">
                <v-switch
                  v-model="settings.disks.invert"
                  :label="$t('plugin_ugreen_leds.invert')"
                  :hint="$t('plugin_ugreen_leds.invert_hint')"
                  persistent-hint inset color="green"
                />
              </v-col>
            </v-row>

            <v-alert
              v-if="mappingDirty"
              type="info" variant="tonal" density="compact" class="mt-3"
              :text="$t('plugin_ugreen_leds.mapping_save_hint')"
            />

            <v-table v-if="detect.mapping.length" density="compact" class="mt-3">
              <thead>
                <tr>
                  <th>{{ $t('plugin_ugreen_leds.slot') }}</th>
                  <th>{{ $t('plugin_ugreen_leds.assignment') }}</th>
                  <th>{{ $t('plugin_ugreen_leds.disk') }}</th>
                  <th style="min-width: 180px">{{ $t('plugin_ugreen_leds.color') }}</th>
                  <th class="text-center" style="width: 60px"></th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="slot in detect.mapping" :key="slot.led">
                  <td>{{ ledLabel(slot.led) }}</td>
                  <td style="min-width: 220px">
                    <v-select
                      :model-value="slotValue(slot.led)"
                      :items="slotItems(slot.led)"
                      density="compact" hide-details variant="underlined"
                      @update:model-value="(v) => setSlotValue(slot.led, v)"
                    />
                  </td>
                  <td>
                    <span v-if="slot.dev">/dev/{{ slot.dev }} <span class="text-medium-emphasis">{{ diskInfo(slot.dev) }}</span></span>
                    <span v-else class="text-medium-emphasis">{{ $t('plugin_ugreen_leds.empty_slot') }}</span>
                  </td>
                  <td>
                    <div class="d-flex align-center ga-1">
                      <ColorField
                        :model-value="settings.disks.per_disk_colors[slot.led] || settings.disks.color_health"
                        @update:model-value="(v) => (settings.disks.per_disk_colors[slot.led] = v)"
                      />
                      <v-btn
                        v-if="settings.disks.per_disk_colors[slot.led]"
                        size="small" icon="mdi-restore" variant="text"
                        @click="resetDiskColor(slot.led)"
                      />
                    </div>
                  </td>
                  <td class="text-center">
                    <v-btn
                      size="small" icon="mdi-lightbulb-on-outline" variant="text"
                      :loading="busy === 'identify_' + slot.led"
                      :title="$t('plugin_ugreen_leds.identify')"
                      @click="identify(slot.led)"
                    />
                  </td>
                </tr>
              </tbody>
            </v-table>

            <v-row class="mt-4">
              <v-col cols="12" md="4">
                <ColorField v-model="settings.disks.color_health" :label="$t('plugin_ugreen_leds.color_health')" />
              </v-col>
              <v-col cols="6" md="4">
                <ColorField v-model="settings.disks.color_standby" :label="$t('plugin_ugreen_leds.color_standby')" />
              </v-col>
              <v-col cols="6" md="4">
                <ColorField v-model="settings.disks.color_unavail" :label="$t('plugin_ugreen_leds.color_unavail')" />
              </v-col>
            </v-row>

            <v-row>
              <v-col cols="12" md="4">
                <v-switch v-model="settings.disks.check_standby" :label="$t('plugin_ugreen_leds.check_standby')" inset color="green" hide-details />
              </v-col>
            </v-row>

          </template>
        </v-card-text>
      </v-card>

      <v-btn color="primary" :loading="saving" @click="saveAndApply">
        <v-icon start>mdi-content-save</v-icon>
        {{ $t('plugin_ugreen_leds.save_apply') }}
      </v-btn>
    </div>

    <v-dialog v-model="logDialog" max-width="900">
      <v-card>
        <v-card-title>{{ $t('plugin_ugreen_leds.log') }}</v-card-title>
        <v-card-text>
          <pre style="white-space: pre-wrap; font-size: 12px; max-height: 60vh; overflow-y: auto">{{ logText || '-' }}</pre>
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn variant="text" color="onPrimary" @click="logDialog = false">{{ $t('plugin_ugreen_leds.close') }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, computed, getCurrentInstance, onMounted, onUnmounted } from 'vue';
import ColorField from './ColorField.vue';

const PLUGIN_NAME = 'ugreen-leds';
const COMMAND = 'ugreen-leds';

const instance = getCurrentInstance();
const t = (key) => instance?.appContext.config.globalProperties.$t(key) ?? key;

const DEFAULTS = {
  enabled: true,
  power: { enabled: true, color: '#ffffff', brightness: 255, mode: 'on', on_ms: 1000, off_ms: 1000 },
  network: {
    enabled: true, interface: 'auto', color: '#ffa500', brightness: 255,
    blink_tx: true, blink_rx: true, blink_interval: 200,
    color_mode: 'static',
    color_100: '#00ff00', color_1000: '#0000ff', color_2500: '#ffff00', color_5000: '#ff00ff', color_10000: '#ffffff',
    dynamic_color_low: '#ff0000', dynamic_color_high: '#00ff00', dynamic_speed_low: 0, dynamic_speed_high: 10000,
    check_gateway: false, color_gateway_unreachable: '#ff0000',
    check_interval: 60,
  },
  disks: {
    enabled: true, mapping: 'ata', slot_map: [],
    brightness: 255, invert: true, refresh_interval: 0.1,
    color_health: '#ffffff', per_disk_colors: {},
    check_standby: true, standby_interval: 1, color_standby: '#0000ff',
    online_interval: 5, color_unavail: '#ff0000',
  },
};

const linkSpeeds = [100, 1000, 2500, 5000, 10000];

const loading = ref(true);
const saving = ref(false);
const busy = ref('');
const logDialog = ref(false);
const logText = ref('');
const statusInterval = ref(null);
const savedMapping = ref('');

const settings = reactive(structuredClone(DEFAULTS));
const status = reactive({ running: false, module_loaded: false, leds: [] });
const detect = reactive({ model: '', method: 'ata', leds: [], mapping: [], disks: [], interfaces: [], default_maps: { ata: [], hctl: [] } });

const powerModes = computed(() => [
  { title: t('plugin_ugreen_leds.mode_on'), value: 'on' },
  { title: t('plugin_ugreen_leds.mode_blink'), value: 'blink' },
  { title: t('plugin_ugreen_leds.mode_breath'), value: 'breath' },
]);

const colorModes = computed(() => [
  { title: t('plugin_ugreen_leds.color_mode_static'), value: 'static' },
  { title: t('plugin_ugreen_leds.color_mode_link_speed'), value: 'link_speed' },
  { title: t('plugin_ugreen_leds.color_mode_dynamic'), value: 'dynamic' },
]);

const mappingMethods = computed(() => [
  { title: t('plugin_ugreen_leds.mapping_ata'), value: 'ata' },
  { title: t('plugin_ugreen_leds.mapping_hctl'), value: 'hctl' },
  { title: t('plugin_ugreen_leds.mapping_serial'), value: 'serial' },
]);

const interfaceItems = computed(() => [
  { title: t('plugin_ugreen_leds.interface_auto'), value: 'auto' },
  ...detect.interfaces.map((i) => ({
    title: `${i.name} (${i.state}${i.speed > 0 ? ', ' + speedLabel(i.speed) : ''})`,
    value: i.name,
  })),
]);

const mappingDirty = computed(() => JSON.stringify({ m: settings.disks.mapping, s: settings.disks.slot_map }) !== savedMapping.value);

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const query = async (args, timeout = 30, parse_json = false) => {
  const res = await fetch('/api/v1/mos/plugins/query', {
    method: 'POST',
    headers: { ...getAuthHeaders(), 'Content-Type': 'application/json' },
    body: JSON.stringify({ command: COMMAND, args, timeout, parse_json }),
  });
  if (!res.ok) throw new Error(`Query ${args[0]} failed`);
  return res.json();
};

const deepMerge = (target, source) => {
  for (const [key, value] of Object.entries(source || {})) {
    if (value && typeof value === 'object' && !Array.isArray(value) && target[key] && typeof target[key] === 'object' && !Array.isArray(target[key])) {
      deepMerge(target[key], value);
    } else {
      target[key] = value;
    }
  }
  return target;
};

const rememberMapping = () => {
  savedMapping.value = JSON.stringify({ m: settings.disks.mapping, s: settings.disks.slot_map });
};

const fetchSettings = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, { headers: getAuthHeaders() });
    if (res.ok) {
      deepMerge(settings, structuredClone(DEFAULTS));
      deepMerge(settings, await res.json());
    }
  } catch (e) {
    console.error('Failed to fetch settings:', e);
  }
  if (!Array.isArray(settings.disks.slot_map)) settings.disks.slot_map = [];
  if (!settings.disks.per_disk_colors || typeof settings.disks.per_disk_colors !== 'object') settings.disks.per_disk_colors = {};
  rememberMapping();
};

const fetchStatus = async () => {
  try {
    const data = await query(['status'], 5, true);
    if (data.success && data.output && typeof data.output === 'object') {
      Object.assign(status, data.output);
    }
  } catch (e) {
    console.error('Failed to fetch status:', e);
  }
};

const fetchDetect = async () => {
  try {
    const data = await query(['detect'], 20, true);
    if (data.success && data.output && typeof data.output === 'object') {
      Object.assign(detect, data.output);
    }
  } catch (e) {
    console.error('Failed to detect hardware:', e);
  }
};

const saveAndApply = async () => {
  saving.value = true;
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: { ...getAuthHeaders(), 'Content-Type': 'application/json' },
      body: JSON.stringify(settings),
    });
    if (!res.ok) throw new Error('Saving settings failed');
    rememberMapping();
    await query(['apply'], 30);
    await Promise.all([fetchStatus(), fetchDetect()]);
  } catch (e) {
    console.error('Failed to save settings:', e);
    alert(t('plugin_ugreen_leds.save_failed'));
  } finally {
    saving.value = false;
  }
};

const runAction = async (action) => {
  busy.value = action;
  try {
    await query([action], 30);
    await fetchStatus();
  } catch (e) {
    console.error(`Failed to ${action}:`, e);
  } finally {
    busy.value = '';
  }
};

const identify = async (led) => {
  busy.value = 'identify_' + led;
  try {
    await query(['identify', led], 20);
    await fetchStatus();
  } catch (e) {
    console.error('Failed to identify LED:', e);
  } finally {
    busy.value = '';
  }
};

const showLog = async () => {
  busy.value = 'log';
  try {
    const data = await query(['log'], 5);
    logText.value = typeof data.output === 'string' ? data.output : JSON.stringify(data.output, null, 2);
    logDialog.value = true;
  } catch (e) {
    console.error('Failed to fetch log:', e);
  } finally {
    busy.value = '';
  }
};

const slotIndex = (led) => parseInt(led.replace('disk', ''), 10) - 1;

const slotValue = (led) => settings.disks.slot_map[slotIndex(led)] || '';

const setSlotValue = (led, value) => {
  const idx = slotIndex(led);
  const map = [...settings.disks.slot_map];
  while (map.length <= idx) map.push('');
  map[idx] = value || '';
  while (map.length && !map[map.length - 1]) map.pop();
  settings.disks.slot_map = map;
};

const slotItems = (led) => {
  const method = settings.disks.mapping;
  const idx = slotIndex(led);
  const items = [];
  if (method === 'serial') {
    items.push({ title: t('plugin_ugreen_leds.unassigned'), value: '' });
  } else {
    const def = detect.default_maps?.[method]?.[idx];
    items.push({ title: `${t('plugin_ugreen_leds.default')}${def ? ' (' + def + ')' : ''}`, value: '' });
  }
  for (const d of detect.disks) {
    const key = d[method];
    if (!key) continue;
    items.push({ title: `${key} – /dev/${d.dev} ${d.model || ''} ${d.size || ''}`.trim(), value: key });
  }
  return items;
};

const resetDiskColor = (led) => {
  delete settings.disks.per_disk_colors[led];
};

const diskInfo = (dev) => {
  const d = detect.disks.find((x) => x.dev === dev);
  if (!d) return '';
  return [d.model, d.serial, d.size].filter(Boolean).join(' · ');
};

const ledLabel = (name) => {
  if (name === 'power') return t('plugin_ugreen_leds.led_power');
  if (name === 'netdev') return t('plugin_ugreen_leds.led_netdev');
  return name.replace('disk', t('plugin_ugreen_leds.led_disk') + ' ');
};

const speedLabel = (speed) => (speed >= 1000 ? `${speed / 1000} Gbit/s` : `${speed} Mbit/s`);

const ledStyle = (led) => {
  const lit = led.state && led.state !== 'off' && led.brightness > 0;
  return {
    width: '22px',
    height: '22px',
    borderRadius: '50%',
    background: lit ? led.color : 'transparent',
    opacity: lit ? Math.max(0.35, led.brightness / 255) : 1,
    border: '2px solid rgba(128, 128, 128, 0.6)',
    boxShadow: lit ? `0 0 8px ${led.color}` : 'none',
  };
};

onMounted(async () => {
  try {
    await fetchSettings();
    await Promise.all([fetchStatus(), fetchDetect()]);
    statusInterval.value = setInterval(fetchStatus, 5000);
  } catch (e) {
    console.error('Failed to initialize:', e);
  } finally {
    loading.value = false;
  }
});

onUnmounted(() => {
  if (statusInterval.value) clearInterval(statusInterval.value);
});
</script>
