<!-- distributions/distributionListFooter.vue -->
<template>
  <div>
    <DistributionSearch
      ref="distributionSearch"
      :active-configuration-type="activeConfigurationType"
      :active-device-model="activeDeviceModel"
      :configurations="configurations"
      :suppls-config-type="eaptlsConfigType"
      @config-type-selected="onConfigTypeSelected"
      @filter-updated="onFilterUpdated"
      @resize="onResize"
      @sort-change="onSortChange"
    />
    <DistributionList
      ref="distributionList"
      :active-configuration-type="activeConfigurationType"
      :sort-opt="sortOpt"
      :filters="filter"
      @error="onError"
    />
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted } from 'vue';
import { ErrorMsgTranslation } from '@header/web-utils';
import { 
  loadConfiguration,
  loadActiveConfiguration,
  loadActiveDeviceModel,
  setActiveConfiguration,
  getEAPTLSConfig
} from '@plugins/configuration-management/util/configurationUtil';
import { configTypes } from '@plugins/configuration-management/common/configTypes';
import { deviceTypes } from '@plugins/configuration-management/common/deviceTypes';
import { permission } from '@plugins/configuration-management/common/configurations';
import type { Configuration, ConfigType, DeviceType, SortOptions } from '@/types/config';

// Async components
const DistributionSearch = defineAsyncComponent(() => 
  import('@plugins/configuration-management/components/distributions/DistributionSearch.vue')
);
const DistributionList = defineAsyncComponent(() =>
  import('@plugins/configuration-management/components/distributions/DistributionList.vue')
);

// Reactive state
const activeConfigurationType = ref<ConfigType>();
const activeDeviceModel = ref<DeviceType>();
const configurations = ref<Configuration[]>([]);
const filter = ref<string>('');
const sortOpt = reactive<SortOptions>({
  sortParam: 'distributionDateTime',
  ascending: false,
});
const observingAlert = ref(false);
const eaptlsConfigType = ref(0);
const distributionList = ref<InstanceType<typeof DistributionList>>();

// Methods
const onConfigTypeSelected = (configType: ConfigType) => {
  setActiveConfiguration(configType);
  activeConfigurationType.value = configType;
};

const onFilterUpdated = (newFilter: string) => {
  filter.value = newFilter;
};

const onError = (error: Error) => {
  const exceptionDetail = ErrorMsgTranslation(error);
  SpublishAlert(
    exceptionDetail.message,
    'error',
    exceptionDetail.detail_message
  );
};

const onSortChange = (newSort: SortOptions) => {
  Object.assign(sortOpt, newSort);
};

const onResize = () => {
  distributionList.value?.onResize();
};

// Alert observer
const handleAlertChange = () => {
  if (observingAlert.value) return;

  const alertObserver = new MutationObserver(() => {
    onResize();
  });

  const alertElement = document.querySelectorAll('[role="alert"]')[0];
  if (alertElement) {
    alertObserver.observe(alertElement, {
      childList: true,
      attributes: true,
      characterData: true,
    });
    observingAlert.value = true;
  }
};

// Lifecycle
onMounted(async () => {
  window.addEventListener('resize', onResize);
  
  const [configs, activeConfig, activeDevice] = await Promise.all([
    loadConfiguration(),
    loadActiveConfiguration(),
    loadActiveDeviceModel()
  ]);

  configurations.value = configs;
  activeConfigurationType.value = activeConfig;
  activeDeviceModel.value = activeDevice;

  if (activeDeviceModel.value) {
    configurations.value = getConfigTypeByDeviceModel(configs, activeDeviceModel.value);
  }

  try {
    const permissions = await bullseyeApi.configManagement.getPermissions();
    if ([permission.ADMINISTRATOR, permission.NETWORK_CONFIG_DISTRIBUTOR].some(p => permissions.includes(p))) {
      if (import.meta.env.VITE_EAPTLS === 'true') {
        eaptlsConfigType.value = await getEAPTLSConfig();
      }
    }
  } catch (error) {
    onError(error as Error);
  }

  handleAlertChange();
});

onUnmounted(() => {
  window.removeEventListener('resize', onResize);
});

// Utility functions
const getConfigTypeByDeviceModel = (configs: Configuration[], deviceModel: DeviceType): Configuration[] => {
  const mapConfigList = [
    {
      type: deviceTypes.SPECTRUM_IO,
      values: [
        configTypes.NETWORK_CONFIGURATION,
        configTypes.DRUG_LIBRARY,
        configTypes.HOL_CERTIFICATE,
        configTypes.WIFI_CONFIGURATION,
      ],
    },
    {
      type: deviceTypes.EVO_IO,
      values: [configTypes.DRUG_LIBRARY],
    },
  ];

  return mapConfigList
    .find(e => e.type === deviceModel)
    ?.values.map(v => configs.find(c => c.type === v))
    .filter(Boolean) as Configuration[];
};
</script>
