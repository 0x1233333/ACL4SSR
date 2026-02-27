```python
def check_clash_builder():
    with open("7sageer/sublink-worker/sublink-worker-6538b828d91fabb17352c4fe4fc1719d9b05fe50/src/builders/ClashConfigBuilder.js", "r") as f:
        print(f.read()[:2000])
check_clash_builder()


```

```text
Traceback (most recent call last):
  File "<string>", line 4, in <module>
  File "<string>", line 2, in check_guide
FileNotFoundError: [Errno 2] No such file or directory: 'uploaded:订阅转换模板定制指南.md'


```

```python
def check_base_builder():
    with open("7sageer/sublink-worker/sublink-worker-6538b828d91fabb17352c4fe4fc1719d9b05fe50/src/builders/BaseConfigBuilder.js", "r") as f:
        print(f.read())
check_base_builder()


```

```text
import { ProxyParser } from '../parsers/index.js';
import { deepCopy, tryDecodeSubscriptionLines, decodeBase64 } from '../utils.js';
import { createTranslator } from '../i18n/index.js';
import { generateRules, getOutbounds, PREDEFINED_RULE_SETS } from '../config/index.js';

export class BaseConfigBuilder {
    constructor(inputString, baseConfig, lang, userAgent, groupByCountry = false, includeAutoSelect = true) {
        this.inputString = inputString;
        this.config = deepCopy(baseConfig);
        this.customRules = [];
        this.selectedRules = [];
        this.t = createTranslator(lang);
        this.userAgent = userAgent;
        this.appliedOverrideKeys = new Set();
        this.groupByCountry = groupByCountry;
        this.includeAutoSelect = includeAutoSelect;
        this.providerUrls = [];  // URLs to use as providers (auto-sync)
    }

    async build() {
        const customItems = await this.parseCustomItems();
        this.addCustomItems(customItems);
        this.addSelectors();
        return this.formatConfig();
    }

    async parseCustomItems() {
        const input = this.inputString || '';
        const parsedItems = [];

        // Import the content parser for direct input parsing
        const { parseSubscriptionContent } = await import('../parsers/subscription/subscriptionContentParser.js');

        // Try to parse the entire input as a config format (Sing-Box JSON or Clash YAML)
        const directResult = parseSubscriptionContent(input);
        if (directResult && typeof directResult === 'object' && directResult.type) {
            // It's a parsed config (singboxConfig or yamlConfig)
            if (directResult.config) {
                this.applyConfigOverrides(directResult.config);
            }
            if (Array.isArray(directResult.proxies)) {
                for (const proxy of directResult.proxies) {
                    if (proxy && proxy.tag) {
                        parsedItems.push(proxy);
                    }
                }
                if (parsedItems.length > 0) return parsedItems;
            }
        }

        // If direct parsing didn't work, check for Base64 encoded content
        const isBase64Like = /^[A-Za-z0-9+/=\r\n]+$/.test(input) && input.replace(/[\r\n]/g, '').length % 4 === 0;
        if (isBase64Like) {
            try {
                const sanitized = input.replace(/\s+/g, '');
                const decodedWhole = decodeBase64(sanitized);
                if (typeof decodedWhole === 'string') {
                    const decodedResult = parseSubscriptionContent(decodedWhole);
                    if (decodedResult && typeof decodedResult === 'object' && decodedResult.type) {
                        if (decodedResult.config) {
                            this.applyConfigOverrides(decodedResult.config);
                        }
                        if (Array.isArray(decodedResult.proxies)) {
                            for (const proxy of decodedResult.proxies) {
                                if (proxy && proxy.tag) {
                                    parsedItems.push(proxy);
                                }
                            }
                            if (parsedItems.length > 0) return parsedItems;
                        }
                    }
                }
            } catch (_) { }
        }

        // Otherwise, line-by-line processing (URLs, subscription content, remote lists, etc.)
        const urls = input.split('\n').filter(url => url.trim() !== '');
        for (const url of urls) {
            let processedUrls = tryDecodeSubscriptionLines(url);
            if (!Array.isArray(processedUrls)) {
                processedUrls = [processedUrls];
            }

            for (const processedUrl of processedUrls) {
                const trimmedUrl = typeof processedUrl === 'string' ? processedUrl.trim() : '';

                // Check if it's an HTTP(S) URL - may use as provider if format matches
                if (trimmedUrl.startsWith('http://') || trimmedUrl.startsWith('https://')) {
                    const { fetchSubscriptionWithFormat } = await import('../parsers/subscription/httpSubscriptionFetcher.js');

                    try {
                        const fetchResult = await fetchSubscriptionWithFormat(trimmedUrl, this.userAgent);
                        if (fetchResult) {
                            const { content, format, url: originalUrl } = fetchResult;

                            // If format is compatible with target client, use as provider
                            if (this.isCompatibleProviderFormat(format)) {
                                this.providerUrls.push(originalUrl);
                                continue;  // Skip parsing, will be used as provider
                            }

                            // Otherwise parse the content as usual
                            const result = parseSubscriptionContent(content);
                            if (result && typeof result === 'object' && (result.type === 'yamlConfig' || result.type === 'singboxConfig' || result.type === 'surgeConfig')) {
                                if (result.config) {
                                    this.applyConfigOverrides(result.config);
                                }
                                if (Array.isArray(result.proxies)) {
                                    result.proxies.forEach(proxy => {
                                        if (proxy && typeof proxy === 'object' && proxy.tag) {
                                            parsedItems.push(proxy);
                                        }
                                    });
                                }
                                continue;
                            }
                            // Handle array of URIs or other formats
                            if (Array.isArray(result)) {
                                for (const item of result) {
                                    if (item && typeof item === 'object' && item.tag) {
                                        parsedItems.push(item);
                                    } else if (typeof item === 'string') {
                                        const subResult = await ProxyParser.parse(item, this.userAgent);
                                        if (subResult) {
                                            parsedItems.push(subResult);
                                        }
                                    }
                                }
                            }
                        }
                    } catch (error) {
                        console.error('Error processing HTTP subscription:', error);
                    }
                    continue;
                }

                // Non-HTTP URLs (protocol URIs like ss://, vmess://, etc.)
                const result = await ProxyParser.parse(processedUrl, this.userAgent);
                // Handle yamlConfig, singboxConfig, and surgeConfig types (they have the same structure)
                if (result && typeof result === 'object' && (result.type === 'yamlConfig' || result.type === 'singboxConfig' || result.type === 'surgeConfig')) {
                    if (result.config) {
                        this.applyConfigOverrides(result.config);
                    }
                    if (Array.isArray(result.proxies)) {
                        result.proxies.forEach(proxy => {
                            if (proxy && typeof proxy === 'object' && proxy.tag) {
                                parsedItems.push(proxy);
                            }
                        });
                    }
                    continue;
                }
                if (Array.isArray(result)) {
                    for (const item of result) {
                        if (item && typeof item === 'object' && item.tag) {
                            parsedItems.push(item);
                        } else if (typeof item === 'string') {
                            const subResult = await ProxyParser.parse(item, this.userAgent);
                            if (subResult) {
                                parsedItems.push(subResult);
                            }
                        }
                    }
                } else if (result) {
                    parsedItems.push(result);
                }
            }
        }

        return parsedItems;
    }

    /**
     * Check if subscription format is compatible for use as a provider
     * Override in child classes to enable provider support
     * @param {'clash'|'singbox'|'unknown'} format - Detected subscription format
     * @returns {boolean} - True if format can be used as provider
     */
    isCompatibleProviderFormat(format) {
        return false;  // Default: no provider support
    }

    applyConfigOverrides(overrides) {
        if (!overrides || typeof overrides !== 'object') {
            return;
        }

        // Block keys that are handled specially:
        // - 'proxies': handled by dedicated parser
        // - 'rules', 'rule-providers': generated by our own logic
        // - 'proxy-groups': stored for later intelligent merge (not direct override)
        const blacklistedKeys = new Set(['proxies', 'rules', 'rule-providers', 'proxy-groups']);

        Object.entries(overrides).forEach(([key, value]) => {
            if (blacklistedKeys.has(key)) {
                return;
            }
            if (value === undefined) {
                delete this.config[key];
                this.appliedOverrideKeys.add(key);
            } else if (key === 'dns' && typeof value === 'object' && !Array.isArray(value)) {
                // Special handling for dns object - merge array fields instead of overwriting
                this.config[key] = this.mergeDnsConfig(this.config[key], value);
                this.appliedOverrideKeys.add(key);
            } else {
                this.config[key] = deepCopy(value);
                this.appliedOverrideKeys.add(key);
            }
        });

        // Store user proxy-groups for later merge (after system groups are created)
        if (Array.isArray(overrides['proxy-groups'])) {
            this.pendingUserProxyGroups = this.pendingUserProxyGroups || [];
            this.pendingUserProxyGroups.push(...overrides['proxy-groups']);
        }
    }

    /**
     * Merge DNS configuration with intelligent array merging
     * Arrays like nameserver, fallback, fake-ip-filter are merged instead of overwritten
     * @param {object} existing - Existing DNS config
     * @param {object} incoming - Incoming DNS config to merge
     * @returns {object} - Merged DNS config
     */
    mergeDnsConfig(existing, incoming) {
        if (!existing || typeof existing !== 'object') {
            return deepCopy(incoming);
        }

        const result = deepCopy(existing);
        // Array fields that should be merged instead of overwritten
        const mergeableArrayKeys = new Set(['nameserver', 'fallback', 'fake-ip-filter']);

        Object.entries(incoming).forEach(([key, value]) => {
            if (mergeableArrayKeys.has(key) && Array.isArray(value)) {
                if (Array.isArray(result[key])) {
                    // Merge arrays and deduplicate
                    result[key] = [...new Set([...result[key], ...value])];
                } else {
                    result[key] = deepCopy(value);
                }
            } else if (key === 'nameserver-policy' && typeof value === 'object' && !Array.isArray(value)) {
                // Merge nameserver-policy object
                result[key] = { ...(result[key] || {}), ...deepCopy(value) };
            } else {
                result[key] = deepCopy(value);
            }
        });

        return result;
    }

    hasConfigOverride(key) {
        return this.appliedOverrideKeys?.has(key);
    }

    getOutboundsList() {
        let outbounds;
        if (typeof this.selectedRules === 'string' && PREDEFINED_RULE_SETS[this.selectedRules]) {
            outbounds = getOutbounds(PREDEFINED_RULE_SETS[this.selectedRules]);
        } else if (this.selectedRules && Object.keys(this.selectedRules).length > 0) {
            outbounds = getOutbounds(this.selectedRules);
        } else {
            outbounds = getOutbounds(PREDEFINED_RULE_SETS.minimal);
        }
        return outbounds;
    }

    getProxyList() {
        return this.getProxies().map(proxy => this.getProxyName(proxy));
    }

    getProxies() {
        throw new Error('getProxies must be implemented in child class');
    }

    getProxyName(proxy) {
        throw new Error('getProxyName must be implemented in child class');
    }

    convertProxy(proxy) {
        throw new Error('convertProxy must be implemented in child class');
    }

    addProxyToConfig(proxy) {
        throw new Error('addProxyToConfig must be implemented in child class');
    }

    addAutoSelectGroup(proxyList) {
        throw new Error('addAutoSelectGroup must be implemented in child class');
    }

    addNodeSelectGroup(proxyList) {
        throw new Error('addNodeSelectGroup must be implemented in child class');
    }

    addOutboundGroups(outbounds, proxyList) {
        throw new Error('addOutboundGroups must be implemented in child class');
    }

    addCustomRuleGroups(proxyList) {
        throw new Error('addCustomRuleGroups must be implemented in child class');
    }

    addFallBackGroup(proxyList) {
        throw new Error('addFallBackGroup must be implemented in child class');
    }

    addCountryGroups() {
        throw new Error('addCountryGroups must be implemented in child class');
    }

    addCustomItems(customItems) {
        const validItems = customItems.filter(item => item != null);
        validItems.forEach(item => {
            if (item?.tag) {
                const convertedProxy = this.convertProxy(item);
                if (convertedProxy) {
                    this.addProxyToConfig(convertedProxy);
                }
            }
        });
    }

    addSelectors() {
        const outbounds = this.getOutboundsList();
        const proxyList = this.getProxyList();

        this.addAutoSelectGroup(proxyList);
        this.addNodeSelectGroup(proxyList);
        if (this.groupByCountry) {
            this.addCountryGroups();
        }
        this.addOutboundGroups(outbounds, proxyList);
        this.addCustomRuleGroups(proxyList);
        this.addFallBackGroup(proxyList);

        // Merge user-defined proxy-groups after system groups are created
        if (this.pendingUserProxyGroups && this.pendingUserProxyGroups.length > 0) {
            this.mergeUserProxyGroups(this.pendingUserProxyGroups);
        }
    }

    /**
     * Merge user-defined proxy groups with system-generated ones
     * Override in child classes to implement format-specific merge logic
     * @param {Array} userGroups - User-defined proxy groups
     */
    mergeUserProxyGroups(userGroups) {
        // Default: no-op. Child classes implement format-specific merge.
    }

    generateRules() {
        return generateRules(this.selectedRules, this.customRules);
    }

    formatConfig() {
        throw new Error('formatConfig must be implemented in child class');
    }
}



```

```python
def check_rules():
    with open("7sageer/sublink-worker/sublink-worker-6538b828d91fabb17352c4fe4fc1719d9b05fe50/src/config/index.js", "r") as f:
        content = f.read()
        import re
        m = re.search(r'export function generateRules.*?\n}', content, re.DOTALL)
        if m:
            print(m.group(0))
        else:
            print(content[:2000])
check_rules()


```

```text
/**
 * Config Module Index
 * Central export point for all configuration modules
 */

// Rule URLs
export {
	SITE_RULE_SET_BASE_URL,
	IP_RULE_SET_BASE_URL,
	CLASH_SITE_RULE_SET_BASE_URL,
	CLASH_IP_RULE_SET_BASE_URL,
	SURGE_SITE_RULE_SET_BASEURL,
	SURGE_IP_RULE_SET_BASEURL
} from './ruleUrls.js';

// Rules
export {
	CUSTOM_RULES,
	UNIFIED_RULES,
	PREDEFINED_RULE_SETS,
	DIRECT_DEFAULT_RULES,
	SITE_RULE_SETS,
	IP_RULE_SETS,
	CLASH_SITE_RULE_SETS,
	CLASH_IP_RULE_SETS
} from './rules.js';

// Rule Generators
export {
	getOutbounds,
	generateRules,
	generateRuleSets,
	generateClashRuleSets
} from './ruleGenerators.js';

// Subconverter Config
export { generateSubconverterConfig } from './subconverterConfig.js';

// Platform Configs
export { SING_BOX_CONFIG, SING_BOX_CONFIG_V1_11 } from './singboxConfig.js';
export { CLASH_CONFIG } from './clashConfig.js';
export { SURGE_CONFIG } from './surgeConfig.js';



```

```python
with open("7sageer/sublink-worker/sublink-worker-6538b828d91fabb17352c4fe4fc1719d9b05fe50/src/config/ruleGenerators.js", "r") as f:
    print(f.read()[:2000])


```

```text
/**
 * Rule Generators
 * Functions for generating rules and rule sets
 */

import { UNIFIED_RULES, PREDEFINED_RULE_SETS, SITE_RULE_SETS, IP_RULE_SETS, CLASH_SITE_RULE_SETS, CLASH_IP_RULE_SETS } from './rules.js';
import { SITE_RULE_SET_BASE_URL, IP_RULE_SET_BASE_URL, CLASH_SITE_RULE_SET_BASE_URL, CLASH_IP_RULE_SET_BASE_URL } from './ruleUrls.js';

// Helper function to get outbounds based on selected rule names
export function getOutbounds(selectedRuleNames) {
	if (!selectedRuleNames || !Array.isArray(selectedRuleNames)) {
		return [];
	}
	return UNIFIED_RULES
		.filter(rule => selectedRuleNames.includes(rule.name))
		.map(rule => rule.name);
}

// Helper function to generate rules based on selected rule names
export function generateRules(selectedRules = [], customRules = []) {
	if (typeof selectedRules === 'string' && PREDEFINED_RULE_SETS[selectedRules]) {
		selectedRules = PREDEFINED_RULE_SETS[selectedRules];
	}

	if (!selectedRules || selectedRules.length === 0) {
		selectedRules = PREDEFINED_RULE_SETS.minimal;
	}

	const rules = [];

	UNIFIED_RULES.forEach(rule => {
		if (selectedRules.includes(rule.name)) {
			rules.push({
				site_rules: rule.site_rules,
				ip_rules: rule.ip_rules,
				domain_suffix: rule?.domain_suffix,
				ip_cidr: rule?.ip_cidr,
				outbound: rule.name
			});
		}
	});

	customRules.reverse();
	customRules.forEach((rule) => {
		rules.unshift({
			site_rules: rule.site ? rule.site.split(',') : [],
			ip_rules: rule.ip ? rule.ip.split(',') : [],
			domain_suffix: rule.domain_suffix ? rule.domain_suffix.split(',') : [],
			domain_keyword: rule.domain_keyword ? rule.domain_keyword.split(',') : [],
			ip_cidr: rule.ip_cidr ? rule.ip_cidr.split(',') : [],
			protocol: rule.protocol ? rule.protocol.split(',') : [],
			outbound: rule.name
		});
	});

	return rules;
}

export function generateRuleSets(selectedRules = [], customRules = []) {
	if (typeof selectedRules === 'string' && PREDEFINED_RULE_SETS[selectedRules]) {
		selectedRules = PREDEFINE


```

```python
with open("7sageer/sublink-worker/sublink-worker-6538b828d91fabb17352c4fe4fc1719d9b05fe50/src/builders/helpers/clashConfigUtils.js", "r") as f:
    print(f.read())


```

```text
export function emitClashRules(rules = [], translator) {
    if (!translator) {
        throw new Error('emitClashRules requires a translator function');
    }
    const results = [];
    rules
        .filter(rule => Array.isArray(rule.domain_suffix) && rule.domain_suffix.length > 0)
        .forEach(rule => {
            rule.domain_suffix.forEach(suffix => {
                results.push(`DOMAIN-SUFFIX,${suffix},${translator('outboundNames.' + rule.outbound)}`);
            });
        });

    rules
        .filter(rule => Array.isArray(rule.domain_keyword) && rule.domain_keyword.length > 0)
        .forEach(rule => {
            rule.domain_keyword.forEach(keyword => {
                results.push(`DOMAIN-KEYWORD,${keyword},${translator('outboundNames.' + rule.outbound)}`);
            });
        });

    rules
        .filter(rule => Array.isArray(rule.site_rules) && rule.site_rules[0])
        .forEach(rule => {
            rule.site_rules.forEach(site => {
                results.push(`RULE-SET,${site},${translator('outboundNames.' + rule.outbound)}`);
            });
        });

    rules
        .filter(rule => Array.isArray(rule.ip_rules) && rule.ip_rules[0])
        .forEach(rule => {
            rule.ip_rules.forEach(ip => {
                results.push(`RULE-SET,${ip}-ip,${translator('outboundNames.' + rule.outbound)},no-resolve`);
            });
        });

    rules
        .filter(rule => Array.isArray(rule.ip_cidr) && rule.ip_cidr.length > 0)
        .forEach(rule => {
            rule.ip_cidr.forEach(cidr => {
                results.push(`IP-CIDR,${cidr},${translator('outboundNames.' + rule.outbound)},no-resolve`);
            });
        });

    return results;
}

const normalize = (s) => typeof s === 'string' ? s.trim() : s;

export function sanitizeClashProxyGroups(config) {
    const groups = config['proxy-groups'] || [];
    if (!Array.isArray(groups) || groups.length === 0) {
        return;
    }
    const proxyNames = new Set((config.proxies || []).map(p => normalize(p?.name)).filter(Boolean));
    const groupNames = new Set(groups.map(g => normalize(g?.name)).filter(Boolean));
    const validNames = new Set(['DIRECT', 'REJECT'].map(normalize));
    proxyNames.forEach(n => validNames.add(n));
    groupNames.forEach(n => validNames.add(n));

    config['proxy-groups'] = groups.map(group => {
        if (!group || !Array.isArray(group.proxies)) return group;
        const normalizedProxies = group.proxies
            .map(x => typeof x === 'string' ? x.trim() : x)
            .filter(x => typeof x === 'string');
        const seen = new Set();
        const deduped = normalizedProxies.filter(value => {
            if (seen.has(value)) return false;
            seen.add(value);
            return true;
        });

        // If group uses providers, we cannot validate provider node names at build time.
        // Skip filtering to avoid incorrectly removing valid provider nodes.
        if (Array.isArray(group.use) && group.use.length > 0) {
            return { ...group, proxies: deduped };
        }

        const filtered = deduped.filter(x => validNames.has(normalize(x)));
        return { ...group, proxies: filtered };
    });
}



```

```python
with open("7sageer/sublink-worker/sublink-worker-6538b828d91fabb17352c4fe4fc1719d9b05fe50/src/config/ruleGenerators.js", "r") as f:
    content = f.read()
    import re
    m = re.search(r'export function generateClashRuleSets.*', content, re.DOTALL)
    if m:
        print(m.group(0)[:2000])



```

```text
export function generateClashRuleSets(selectedRules = [], customRules = [], useMrs = true) {
	if (typeof selectedRules === 'string' && PREDEFINED_RULE_SETS[selectedRules]) {
		selectedRules = PREDEFINED_RULE_SETS[selectedRules];
	}

	if (!selectedRules || selectedRules.length === 0) {
		selectedRules = PREDEFINED_RULE_SETS.minimal;
	}

	// Determine format based on client compatibility
	const format = useMrs ? 'mrs' : 'yaml';
	const ext = useMrs ? '.mrs' : '.yaml';

	const selectedRulesSet = new Set(selectedRules);

	const siteRuleSets = new Set();
	const ipRuleSets = new Set();

	UNIFIED_RULES.forEach(rule => {
		if (selectedRulesSet.has(rule.name)) {
			rule.site_rules.forEach(siteRule => siteRuleSets.add(siteRule));
			rule.ip_rules.forEach(ipRule => ipRuleSets.add(ipRule));
		}
	});

	const site_rule_providers = {};
	const ip_rule_providers = {};

	Array.from(siteRuleSets).forEach(rule => {
		site_rule_providers[rule] = {
			type: 'http',
			format: format,
			behavior: 'domain',
			url: `${CLASH_SITE_RULE_SET_BASE_URL}${rule}${ext}`,
			path: `./ruleset/${rule}${ext}`,
			interval: 86400
		};
	});

	Array.from(ipRuleSets).forEach(rule => {
		ip_rule_providers[`${rule}-ip`] = {
			type: 'http',
			format: format,
			behavior: 'ipcidr',
			url: `${CLASH_IP_RULE_SET_BASE_URL}${rule}${ext}`,
			path: `./ruleset/${rule}-ip${ext}`,
			interval: 86400
		};
	});

	// Add Non-China rule set if not included
	if (!selectedRules.includes('Non-China')) {
		site_rule_providers['geolocation-!cn'] = {
			type: 'http',
			format: format,
			behavior: 'domain',
			url: `${CLASH_SITE_RULE_SET_BASE_URL}geolocation-!cn${ext}`,
			path: `./ruleset/geolocation-!cn${ext}`,
			interval: 86400
		};
	}

	// Add custom rules
	if (customRules) {
		customRules.forEach(rule => {
			if (rule.site && rule.site != '') {
				rule.site.split(',').forEach(site => {
					const site_trimmed = site.trim();
					site_rule_providers[site_trimmed] = {
						type: 'http',
						format: format,
						beha


```

```python
with open("7sageer/sublink-worker/sublink-worker-6538b828d91fabb17352c4fe4fc1719d9b05fe50/src/config/rules.js", "r") as f:
    content = f.read()
    import re
    m = re.search(r'export const UNIFIED_RULES = \[.*?\n\];', content, re.DOTALL)
    if m:
        print(m.group(0)[:2000])


```

```text
export const UNIFIED_RULES = [
	{
		name: 'Ad Block',
		site_rules: ['category-ads-all'],
		ip_rules: []
	},
	{
		name: 'AI Services',
		site_rules: ['category-ai-!cn',],
		ip_rules: []
	},
	{
		name: 'Bilibili',
		site_rules: ['bilibili'],
		ip_rules: []
	},
	{
		name: 'Youtube',
		site_rules: ['youtube'],
		ip_rules: []
	},
	{
		name: 'Google',
		site_rules: ['google'],
		ip_rules: ['google']
	},
	{
		name: 'Private',
		site_rules: [],
		ip_rules: ['private']
	},
	{
		name: 'Location:CN',
		site_rules: ['geolocation-cn', 'cn'],
		ip_rules: ['cn']
	},
	{
		name: 'Telegram',
		site_rules: [],
		ip_rules: ['telegram']
	},
	{
		name: 'Github',
		site_rules: ['github', 'gitlab'],
		ip_rules: []
	},
	{
		name: 'Microsoft',
		site_rules: ['microsoft'],
		ip_rules: []
	},
	{
		name: 'Apple',
		site_rules: ['apple'],
		ip_rules: []
	},
	{
		name: 'Social Media',
		site_rules: ['facebook', 'instagram', 'twitter', 'tiktok', 'linkedin'],
		ip_rules: []
	},
	{
		name: 'Streaming',
		site_rules: ['netflix', 'hulu', 'disney', 'hbo', 'amazon', 'bahamut'],
		ip_rules: []
	},
	{
		name: 'Gaming',
		site_rules: ['steam', 'epicgames', 'ea', 'ubisoft', 'blizzard'],
		ip_rules: []
	},
	{
		name: 'Education',
		site_rules: ['coursera', 'edx', 'udemy', 'khanacademy', 'category-scholar-!cn'],
		ip_rules: []
	},
	{
		name: 'Financial',
		site_rules: ['paypal', 'visa', 'mastercard', 'stripe', 'wise'],
		ip_rules: []
	},
	{
		name: 'Cloud Services',
		site_rules: ['aws', 'azure', 'digitalocean', 'heroku', 'dropbox'],
		ip_rules: []
	},
	{
		name: 'Non-China',
		site_rules: ['geolocation-!cn'],
		ip_rules: []
	}
];


```

```python
with open("7sageer/sublink-worker/sublink-worker-6538b828d91fabb17352c4fe4fc1719d9b05fe50/src/config/rules.js", "r") as f:
    content = f.read()
    import re
    m = re.search(r'export const PREDEFINED_RULE_SETS = \{.*?\}', content, re.DOTALL)
    if m:
        print(m.group(0)[:2000])


```

```text
export const PREDEFINED_RULE_SETS = {
	minimal: ['Location:CN', 'Private', 'Non-China'],
	balanced: ['Location:CN', 'Private', 'Non-China', 'Github', 'Google', 'Youtube', 'AI Services', 'Telegram'],
	comprehensive: UNIFIED_RULES.map(rule => rule.name)
}


```

```python
with open("7sageer/sublink-worker/sublink-worker-6538b828d91fabb17352c4fe4fc1719d9b05fe50/src/i18n/index.js", "r") as f:
    content = f.read()
    print(content)


```

```text
import { APP_NAME } from '../constants.js';
import { checkStartsWith } from "../utils.js";

// 定义语言包
export const translations = {
  'zh-CN': {
    enableClashUI: '启用 Clash API',
    enableClashUITip: '在 SingBox 配置中启用 Clash API (支持 Dashboard 面板)，默认端口 9090',
    externalController: '外部控制端口',
    externalControllerPlaceholder: '默认 0.0.0.0:9090',
    externalUiDownloadUrl: 'Clash UI 下载地址',
    externalUiDownloadUrlPlaceholder: '默认 zashboard 仓库压缩包地址',
    missingInput: '缺少输入参数',
    missingConfig: '缺少配置参数',
    missingUrl: '缺少URL参数',
    shortUrlNotFound: '短链接未找到',
    invalidShortUrl: '无效的短链接',
    urlParsedSuccess: '已成功解析订阅链接配置',
    internalError: '内部服务器错误',
    notFound: '未找到',
    invalidFormat: '无效格式：',
    defaultRules: ['广告拦截', '谷歌服务', '国外媒体', '电报消息'],
    configValidationError: '配置验证错误：',
    pageDescription: `${APP_NAME} - 订阅链接转换工具`,
    pageKeywords: '订阅链接,转换,Xray,SingBox,Clash,Surge',
    pageTitle: `${APP_NAME} - 订阅链接转换工具`,
    ogTitle: `${APP_NAME} - 订阅链接转换工具`,
    ogDescription: '一个强大的订阅链接转换工具，支持多种客户端格式',
    shareUrls: '输入源',
    urlPlaceholder: '支持粘贴：分享链接、Clash 配置、Sing-Box 配置、Surge 配置...',
    advancedOptions: '高级选项',
    baseConfigSettings: '基础配置设置',
    baseConfigTooltip: '在此处自定义您的基础配置',
    saveConfig: '保存配置',
    savingConfig: '保存中...',
    configContentRequired: '请先输入基础配置内容',
    clearConfig: '清除配置',
    convert: '转换',
    clear: '清除',
    paste: '粘贴',
    processing: '处理中...',
    errorGeneratingLinks: '生成链接时出错',
    confirmClearConfig: '确定要清除配置吗？',
    confirmClearAll: '确定要清除所有内容吗？',
    saveConfigSuccess: '配置保存成功！',
    customPath: '自定义路径',
    savedPaths: '已保存的路径',
    shortenLinks: '生成短链接',
    ruleSelection: '规则选择',
    ruleSelectionTooltip: '选择您需要的规则集',
    copySubconverterUrl: '复制配置地址',
    copiedSubconverterUrl: '已复制！',
    subconverterConfigTitle: 'Subconverter 外部配置',
    subconverterConfigDesc: '根据上方选择的规则和设置，生成 Subconverter 外部配置地址，可直接用于 Subconverter 的 config 参数。',
    custom: '自定义',
    minimal: '最小化',
    balanced: '均衡',
    comprehensive: '全面',
    addCustomRule: '添加自定义规则',
    customRuleOutboundName: '出站名称*',
    customRuleGeoSite: 'Geo-Site规则集',
    customRuleGeoSiteTooltip: 'SingBox中的Site规则来自 https://github.com/lyc8503/sing-box-rules，这意味着您的自定义规则必须在该仓库中',
    customRuleGeoSitePlaceholder: '例如：google,anthropic',
    customRuleGeoIP: 'Geo-IP规则集',
    customRuleGeoIPTooltip: 'SingBox中的IP规则来自 https://github.com/lyc8503/sing-box-rules，这意味着您的自定义规则必须在该仓库中',
    customRuleGeoIPPlaceholder: '例如：private,cn',
    customRuleDomainSuffix: '域名后缀',
    customRuleDomainSuffixPlaceholder: '域名后缀（用逗号分隔）',
    customRuleDomainKeyword: '域名关键词',
    customRuleDomainKeywordPlaceholder: '域名关键词（用逗号分隔）',
    customRuleIPCIDR: 'IP CIDR',
    customRuleIPCIDRPlaceholder: 'IP CIDR（用逗号分隔）',
    customRuleProtocol: '协议类型',
    customRuleProtocolTooltip: '特定流量类型的协议规则。更多详情：https://sing-box.sagernet.org/configuration/route/sniff/',
    customRuleProtocolPlaceholder: '协议（用逗号分隔，例如：http,ssh,dns）',
    removeCustomRule: '移除',
    addCustomRuleJSON: '添加JSON规则',
    customRuleJSON: 'JSON规则',
    customRuleJSONTooltip: '使用JSON格式添加自定义规则，支持批量添加',
    customRulesSection: '自定义规则',
    customRulesSectionTooltip: '创建自定义路由规则来控制特定流量的路由行为。',
    customRulesForm: '表单视图',
    customRulesJSON: 'JSON视图',
    customRule: '自定义规则',
    convertToJSON: '转换为JSON',
    convertToForm: '转换为表单',
    validateJSON: '验证JSON',
    validateConfig: '验证配置',
    validJsonConfig: 'JSON 配置格式有效',
    validYamlConfig: 'YAML 配置格式有效',
    parserUnavailable: '解析器未就绪，请刷新后重试',
    clearAll: '清空所有',
    addJSONRule: '添加JSON规则',
    noCustomRulesForm: '点击"添加自定义规则"开始创建规则',
    noCustomRulesJSON: '点击"添加JSON规则"开始创建规则',
    confirmClearAllRules: '确定要清空所有自定义规则吗？',
    noFormRulesToConvert: '没有表单规则可以转换',
    noValidJSONToConvert: '没有有效的JSON规则可以转换',
    convertedFromForm: '从表单转换',
    convertedFromJSON: '从JSON转换',
    mustBeArray: '必须是数组格式',
    nameRequired: '规则名称是必需的',
    invalidJSON: '无效的JSON格式',
    allJSONValid: '所有JSON规则都有效！',
    jsonValidationErrors: 'JSON验证错误',
    // 规则名称和出站名称的翻译
    outboundNames: {
      'Auto Select': '⚡ 自动选择',
      'Node Select': '🚀 节点选择',
      'Fall Back': '🐟 漏网之鱼',
      'Ad Block': '🛑 广告拦截',
      'AI Services': '💬 AI 服务',
      'Bilibili': '📺 哔哩哔哩',
      'Youtube': '📹 油管视频',
      'Google': '🔍 谷歌服务',
      'Private': '🏠 私有网络',
      'Location:CN': '🔒 国内服务',
      'Telegram': '📲 电报消息',
      'Github': '🐱 Github',
      'Microsoft': 'Ⓜ️ 微软服务',
      'Apple': '🍏 苹果服务',
      'Social Media': '🌐 社交媒体',
      'Streaming': '🎬 流媒体',
      'Gaming': '🎮 游戏平台',
      'Education': '📚 教育资源',
      'Financial': '💰 金融服务',
      'Cloud Services': '☁️ 云服务',
      'Non-China': '🌐 非中国',
      'Manual Switch': '🖐️ 手动切换',
      'GLOBAL': 'GLOBAL'
    },
    generalSettings: '通用设置',
    groupByCountry: '按国家分组',
    groupByCountryTip: '仅 Clash/Surge/SingBox 生效',
    includeAutoSelect: '包含自动选择分组',
    UASettings: '自定义UserAgent',
    UAtip: '默认值curl/7.74.0',
    subscriptionLinks: '订阅链接',
    xrayLink: 'Xray 链接 (Base64)',
    singboxLink: 'SingBox 链接',
    clashLink: 'Clash 链接',
    surgeLink: 'Surge 链接',
    copied: '已复制！',
    shortening: '正在生成短链接...',
    alreadyShortened: '链接已经是短链接了！',
    shortenFailed: '生成短链接失败，请重试',
    customShortCode: '自定义短链接代码',
    optional: '可选',
    customShortCodePlaceholder: '留空自动生成，或输入自定义代码',
    showFullLinks: '显示完整链接',
    noLinkProvided: '没有提供链接！',
    scanQRCode: '扫描二维码',
    tryShortLinks: '请尝试使用短链接！',
    configSaved: '配置保存成功！',
    configSaveFailed: '保存配置失败',
    error: '错误：',
    validJSON: '有效的JSON',
    rules: '条规则',
    rule: '规则',
    // UpdateChecker
    newVersionAvailable: '发现新版本',
    viewRelease: '查看更新',
    updateGuide: '更新指南',
    later: '稍后'
  },
  'en-US': {
    enableClashUI: 'Enable Clash API',
    enableClashUITip: 'Enable Clash API in SingBox config (Support Dashboard), default port 9090',
    externalController: 'External Controller',
    externalControllerPlaceholder: 'Default 0.0.0.0:9090',
    externalUiDownloadUrl: 'External UI Download URL',
    externalUiDownloadUrlPlaceholder: 'Default zashboard archive URL',
    missingInput: 'Missing input parameter',
    missingConfig: 'Missing config parameter',
    missingUrl: 'Missing URL parameter',
    shortUrlNotFound: 'Short URL not found',
    invalidShortUrl: 'Invalid short URL',
    urlParsedSuccess: 'Successfully parsed subscription link configuration',
    internalError: 'Internal Server Error',
    notFound: 'Not Found',
    invalidFormat: 'Invalid format: ',
    defaultRules: ['Ad Blocking', 'Google Services', 'Foreign Media', 'Telegram'],
    configValidationError: 'Config validation error: ',
    pageDescription: `${APP_NAME} - Subscription Link Converter`,
    pageKeywords: 'subscription link,converter,Xray,SingBox,Clash,Surge',
    pageTitle: `${APP_NAME} - Subscription Link Converter`,
    ogTitle: `${APP_NAME} - Subscription Link Converter`,
    ogDescription: 'A powerful subscription link converter supporting multiple client formats',
    shareUrls: 'Input Source',
    urlPlaceholder: 'Paste share links, Clash config, Sing-Box config, or Surge config...',
    advancedOptions: 'Advanced Options',
    baseConfigSettings: 'Base Config Settings',
    baseConfigTooltip: 'Customize your base configuration here',
    saveConfig: 'Save Config',
    savingConfig: 'Saving...',
    configContentRequired: 'Please enter base config content first',
    clearConfig: 'Clear Config',
    convert: 'Convert',
    clear: 'Clear',
    paste: 'Paste',
    processing: 'Processing...',
    errorGeneratingLinks: 'Error generating links',
    confirmClearConfig: 'Are you sure you want to clear the configuration?',
    confirmClearAll: 'Are you sure you want to clear all?',
    saveConfigSuccess: 'Configuration saved successfully!',
    customPath: 'Custom Path',
    savedPaths: 'Saved Paths',
    shortenLinks: 'Generate Short Links',
    ruleSelection: 'Rule Selection',
    ruleSelectionTooltip: 'Select your desired rule sets',
    copySubconverterUrl: 'Copy Config URL',
    copiedSubconverterUrl: 'Copied!',
    subconverterConfigTitle: 'Subconverter External Config',
    subconverterConfigDesc: 'Generate a Subconverter external config URL based on the rules and settings above, for use as the config parameter in Subconverter.',
    custom: 'Custom',
    minimal: 'Minimal',
    balanced: 'Balanced',
    comprehensive: 'Comprehensive',
    addCustomRule: 'Add Custom Rule',
    customRuleOutboundName: 'Outbound Name*',
    customRuleGeoSite: 'Geo-Site Rules',
    customRuleGeoSiteTooltip: 'SingBox Site rules come from https://github.com/lyc8503/sing-box-rules, which means your custom rules must be in that repository',
    customRuleGeoSitePlaceholder: 'e.g., google,anthropic',
    customRuleGeoIP: 'Geo-IP Rules',
    customRuleGeoIPTooltip: 'SingBox IP rules come from https://github.com/lyc8503/sing-box-rules, which means your custom rules must be in that repository',
    customRuleGeoIPPlaceholder: 'e.g., private,cn',
    customRuleDomainSuffix: 'Domain Suffix',
    customRuleDomainSuffixPlaceholder: 'Domain suffixes (comma separated)',
    customRuleDomainKeyword: 'Domain Keyword',
    customRuleDomainKeywordPlaceholder: 'Domain keywords (comma separated)',
    customRuleIPCIDR: 'IP CIDR',
    customRuleIPCIDRPlaceholder: 'IP CIDR (comma separated)',
    customRuleProtocol: 'Protocol Type',
    customRuleProtocolTooltip: 'Protocol rules for specific traffic types. More details: https://sing-box.sagernet.org/configuration/route/sniff/',
    customRuleProtocolPlaceholder: 'Protocols (comma separated, e.g., http,ssh,dns)',
    removeCustomRule: 'Remove',
    addCustomRuleJSON: 'Add JSON Rule',
    customRuleJSON: 'JSON Rule',
    customRuleJSONTooltip: 'Add custom rules using JSON format, supports batch adding',
    customRulesSection: 'Custom Rules',
    customRulesSectionTooltip: 'Create custom routing rules to control traffic routing behavior. Supports both form and JSON editing modes with bidirectional conversion.',
    customRulesForm: 'Form View',
    customRulesJSON: 'JSON View',
    customRule: 'Custom Rule',
    convertToJSON: 'Convert to JSON',
    convertToForm: 'Convert to Form',
    validateJSON: 'Validate JSON',
    validateConfig: 'Validate Config',
    validJsonConfig: 'JSON config is valid',
    validYamlConfig: 'YAML config is valid',
    parserUnavailable: 'Parser unavailable. Please refresh and try again.',
    clearAll: 'Clear All',
    addJSONRule: 'Add JSON Rule',
    noCustomRulesForm: 'Click "Add Custom Rule" to start creating rules',
    noCustomRulesJSON: 'Click "Add JSON Rule" to start creating rules',
    confirmClearAllRules: 'Are you sure you want to clear all custom rules?',
    noFormRulesToConvert: 'No form rules to convert',
    noValidJSONToConvert: 'No valid JSON rules to convert',
    convertedFromForm: 'Converted from Form',
    convertedFromJSON: 'Converted from JSON',
    mustBeArray: 'Must be an array format',
    nameRequired: 'Rule name is required',
    invalidJSON: 'Invalid JSON format',
    allJSONValid: 'All JSON rules are valid!',
    jsonValidationErrors: 'JSON validation errors',
    outboundNames: {
      'Auto Select': '⚡ Auto Select',
      'Node Select': '🚀 Node Select',
      'Fall Back': '🐟 Fall Back',
      'Ad Block': '🛑 Ad Blocking',
      'AI Services': '💬 AI Services',
      'Bilibili': '📺 Bilibili',
      'Youtube': '📹 Youtube',
      'Google': '🔍 Google Services',
      'Private': '🏠 Private Network',
      'Location:CN': '🔒 China Services',
      'Telegram': '📲 Telegram',
      'Github': '🐱 Github',
      'Microsoft': 'Ⓜ️ Microsoft Services',
      'Apple': '🍏 Apple Services',
      'Social Media': '🌐 Social Media',
      'Streaming': '🎬 Streaming',
      'Gaming': '🎮 Gaming Platform',
      'Education': '📚 Education Resources',
      'Financial': '💰 Financial Services',
      'Cloud Services': '☁️ Cloud Services',
      'Non-China': '🌐 Non-China',
      'Manual Switch': '🖐️ Manual Switch',
      'GLOBAL': 'GLOBAL'
    },
    generalSettings: 'General Settings',
    groupByCountry: 'Group by Country',
    groupByCountryTip: 'Clash/Surge/SingBox only',
    includeAutoSelect: 'Include Auto Select Group',
    UASettings: 'Custom UserAgent',
    UAtip: 'By default it will use curl/7.74.0',
    subscriptionLinks: 'Subscription Links',
    xrayLink: 'Xray Link (Base64)',
    singboxLink: 'SingBox Link',
    clashLink: 'Clash Link',
    surgeLink: 'Surge Link',
    copied: 'Copied!',
    shortening: 'Shortening...',
    alreadyShortened: 'Links are already shortened!',
    shortenFailed: 'Failed to shorten URLs. Please try again.',
    customShortCode: 'Custom Short Code',
    optional: 'Optional',
    customShortCodePlaceholder: 'Leave empty for auto-generation, or enter custom code',
    showFullLinks: 'Show Full Links',
    noLinkProvided: 'No link provided!',
    scanQRCode: 'Scan QR Code',
    tryShortLinks: 'Try to use short links!',
    configSaved: 'Configuration saved successfully!',
    configSaveFailed: 'Failed to save configuration',
    error: 'Error: ',
    validJSON: 'Valid JSON',
    rules: 'rules',
    rule: 'Rule',
    // UpdateChecker
    newVersionAvailable: 'New Version Available',
    viewRelease: 'View Release',
    updateGuide: 'Update Guide',
    later: 'Later'
  },
  'fa': {
    missingInput: 'پارامتر ورودی وجود ندارد',
    missingConfig: 'پارامتر پیکربندی وجود ندارد',
    missingUrl: 'پارامتر URL وجود ندارد',
    shortUrlNotFound: 'لینک کوتاه پیدا نشد',
    invalidShortUrl: 'لینک کوتاه نامعتبر',
    urlParsedSuccess: 'پیکربندی لینک اشتراک با موفقیت تجزیه شد',
    internalError: 'خطای داخلی سرور',
    notFound: 'یافت نشد',
    invalidFormat: 'فرمت نامعتبر: ',
    defaultRules: ['مسدودسازی تبلیغات', 'سرویس‌های گوگل', 'رسانه‌های خارجی', 'تلگرام'],
    configValidationError: 'خطای اعتبارسنجی پیکربندی: ',
    pageDescription: `${APP_NAME} - مبدل لینک اشتراک`,
    pageKeywords: 'لینک اشتراک,مبدل,Xray,SingBox,Clash,Surge',
    pageTitle: `${APP_NAME} - مبدل لینک اشتراک`,
    ogTitle: `${APP_NAME} - مبدل لینک اشتراک`,
    ogDescription: 'یک مبدل قدرتمند لینک اشتراک با پشتیبانی از فرمت‌های مختلف',
    shareUrls: 'منبع ورودی',
    urlPlaceholder: 'لینک‌های اشتراک، پیکربندی Clash، Sing-Box یا Surge را جایگذاری کنید...',
    advancedOptions: 'گزینه‌های پیشرفته',
    baseConfigSettings: 'تنظیمات پیکربندی پایه',
    baseConfigTooltip: 'پیکربندی پایه خود را اینجا سفارشی کنید',
    saveConfig: 'ذخیره پیکربندی',
    savingConfig: 'در حال ذخیره...',
    configContentRequired: 'لطفاً ابتدا محتوای پیکربندی پایه را وارد کنید',
    clearConfig: 'پاک کردن پیکربندی',
    convert: 'تبدیل',
    clear: 'پاک کردن',
    paste: 'چسباندن',
    processing: 'در حال پردازش...',
    errorGeneratingLinks: 'خطا در ایجاد لینک‌ها',
    confirmClearConfig: 'آیا مطمئن هستید که می‌خواهید پیکربندی را پاک کنید؟',
    confirmClearAll: 'آیا مطمئن هستید که می‌خواهید همه را پاک کنید؟',
    saveConfigSuccess: 'پیکربندی با موفقیت ذخیره شد!',
    customPath: 'مسیر سفارشی',
    savedPaths: 'مسیرهای ذخیره شده',
    shortenLinks: 'ایجاد لینک‌های کوتاه',
    ruleSelection: 'انتخاب قوانین',
    ruleSelectionTooltip: 'مجموعه قوانین مورد نظر خود را انتخاب کنید',
    copySubconverterUrl: 'کپی آدرس پیکربندی',
    copiedSubconverterUrl: 'کپی شد!',
    subconverterConfigTitle: 'پیکربندی خارجی Subconverter',
    subconverterConfigDesc: 'آدرس پیکربندی خارجی Subconverter را بر اساس قوانین و تنظیمات بالا ایجاد کنید.',
    custom: 'سفارشی',
    minimal: 'حداقل',
    balanced: 'متعادل',
    comprehensive: 'جامع',
    addCustomRule: 'افزودن قانون سفارشی',
    customRuleOutboundName: 'نام خروجی*',
    customRuleGeoSite: 'قوانین Geo-Site',
    customRuleGeoSiteTooltip: 'قوانین SingBox Site از https://github.com/lyc8503/sing-box-rules می‌آیند، به این معنی که قوانین سفارشی شما باید در آن مخزن باشد',
    customRuleGeoSitePlaceholder: 'برای مثال: google,anthropic',
    customRuleGeoIP: 'قوانین Geo-IP',
    customRuleGeoIPTooltip: 'قوانین SingBox IP از https://github.com/lyc8503/sing-box-rules می‌آیند، به این معنی که قوانین سفارشی شما باید در آن مخزن باشد',
    customRuleGeoIPPlaceholder: 'برای مثال: private,cn',
    customRuleDomainSuffix: 'پسوند دامنه',
    customRuleDomainSuffixPlaceholder: 'پسوندهای دامنه (با کاما جدا شده)',
    customRuleDomainKeyword: 'کلمه کلیدی دامنه',
    customRuleDomainKeywordPlaceholder: 'کلمات کلیدی دامنه (با کاما جدا شده)',
    customRuleIPCIDR: 'IP CIDR',
    customRuleIPCIDRPlaceholder: 'IP CIDR (با کاما جدا شده)',
    customRuleProtocol: 'نوع پروتکل',
    customRuleProtocolTooltip: 'قوانین پروتکل برای انواع خاص ترافیک. جزئیات بیشتر: https://sing-box.sagernet.org/configuration/route/sniff/',
    customRuleProtocolPlaceholder: 'پروتکل‌ها (با کاما جدا شده، مثلاً: http,ssh,dns)',
    removeCustomRule: 'حذف',
    addCustomRuleJSON: 'افزودن قانون JSON',
    customRuleJSON: 'قانون JSON',
    customRuleJSONTooltip: 'افزودن قوانین سفارشی با استفاده از فرمت JSON، پشتیبانی از افزودن دسته‌ای',
    customRulesSection: 'قوانین سفارشی',
    customRulesSectionTooltip: 'قوانین مسیریابی سفارشی برای کنترل رفتار مسیریابی ترافیک ایجاد کنید. از حالت‌های ویرایش فرم و JSON با تبدیل دوطرفه پشتیبانی می‌کند.',
    customRulesForm: 'نمای فرم',
    customRulesJSON: 'نمای JSON',
    customRule: 'قانون سفارشی',
    convertToJSON: 'تبدیل به JSON',
    convertToForm: 'تبدیل به فرم',
    validateJSON: 'اعتبارسنجی JSON',
    validateConfig: 'اعتبارسنجی پیکربندی',
    validJsonConfig: 'پیکربندی JSON معتبر است',
    validYamlConfig: 'پیکربندی YAML معتبر است',
    parserUnavailable: 'تجزیه\u200cگر در دسترس نیست. لطفاً صفحه را تازه\u200cسازی کنید.',
    clearAll: 'پاک کردن همه',
    addJSONRule: 'افزودن قانون JSON',
    noCustomRulesForm: 'روی "افزودن قانون سفارشی" کلیک کنید تا شروع به ایجاد قوانین کنید',
    noCustomRulesJSON: 'روی "افزودن قانون JSON" کلیک کنید تا شروع به ایجاد قوانین کنید',
    confirmClearAllRules: 'آیا مطمئن هستید که می‌خواهید همه قوانین سفارشی را پاک کنید؟',
    noFormRulesToConvert: 'هیچ قانون فرمی برای تبدیل وجود ندارد',
    noValidJSONToConvert: 'هیچ قانون JSON معتبری برای تبدیل وجود ندارد',
    convertedFromForm: 'از فرم تبدیل شده',
    convertedFromJSON: 'از JSON تبدیل شده',
    mustBeArray: 'باید در قالب آرایه باشد',
    nameRequired: 'نام قانون الزامی است',
    invalidJSON: 'فرمت JSON نامعتبر',
    allJSONValid: 'همه قوانین JSON معتبر هستند!',
    jsonValidationErrors: 'خطاهای اعتبارسنجی JSON',
    outboundNames: {
      'Auto Select': '⚡ انتخاب خودکار',
      'Node Select': '🚀 انتخاب نود',
      'Fall Back': '🐟 فال بک',
      'Ad Block': '🛑 مسدودسازی تبلیغات',
      'AI Services': '💬 سرویس‌های هوش مصنوعی',
      'Bilibili': '📺 بیلی‌بیلی',
      'Youtube': '📹 یوتیوب',
      'Google': '🔍 سرویس‌های گوگل',
      'Private': '🏠 شبکه خصوصی',
      'Location:CN': '🔒 سرویس‌های چین',
      'Telegram': '📲 تلگرام',
      'Github': '🐱 گیت‌هاب',
      'Microsoft': 'Ⓜ️ سرویس‌های مایکروسافت',
      'Apple': '🍏 سرویس‌های اپل',
      'Social Media': '🌐 شبکه‌های اجتماعی',
      'Streaming': '🎬 استریمینگ',
      'Gaming': '🎮 پلتفرم بازی',
      'Education': '📚 منابع آموزشی',
      'Financial': '💰 سرویس‌های مالی',
      'Cloud Services': '☁️ سرویس‌های ابری',
      'Non-China': '🌐 خارج از چین',
      'Manual Switch': '🖐️ Manual Switch',
      'GLOBAL': 'GLOBAL'
    },
    generalSettings: 'تنظیمات عمومی',
    groupByCountry: 'گروه‌بندی بر اساس کشور',
    groupByCountryTip: 'فقط Clash/Surge/SingBox',
    includeAutoSelect: 'شامل گروه انتخاب خودکار',
    UASettings: 'UserAgent سفارشی',
    UAtip: 'به طور پیش‌فرض از curl/7.74.0 استفاده می‌کند',
    subscriptionLinks: 'لینک‌های اشتراک',
    xrayLink: 'لینک Xray (Base64)',
    singboxLink: 'لینک SingBox',
    clashLink: 'لینک Clash',
    surgeLink: 'لینک Surge',
    copied: 'کپی شد!',
    shortening: 'در حال کوتاه کردن...',
    alreadyShortened: 'لینک‌ها از قبل کوتاه شده‌اند!',
    shortenFailed: 'کوتاه کردن URL ناموفق بود. لطفاً دوباره تلاش کنید.',
    customShortCode: 'کد کوتاه سفارشی',
    optional: 'اختیاری',
    customShortCodePlaceholder: 'برای تولید خودکار خالی بگذارید یا کد سفارشی وارد کنید',
    showFullLinks: 'نمایش لینک‌های کامل',
    noLinkProvided: 'لینکی ارائه نشده است!',
    scanQRCode: 'اسکن کد QR',
    tryShortLinks: 'لطفاً از لینک‌های کوتاه استفاده کنید!',
    configSaved: 'پیکربندی با موفقیت ذخیره شد!',
    configSaveFailed: 'ذخیره پیکربندی ناموفق بود',
    error: 'خطا: ',
    validJSON: 'JSON معتبر',
   ...

```

```python
with open("7sageer/sublink-worker/sublink-worker-6538b828d91fabb17352c4fe4fc1719d9b05fe50/src/i18n/index.js", "r") as f:
    content = f.read()
    import re
    m = re.search(r'export function createTranslator.*', content, re.DOTALL)
    if m:
        print(m.group(0)[:500])


```

```text
export function createTranslator(lang) {
  const currentLang = resolveLanguage(lang);

  return function t(key) {
    const keys = key.split('.');
    let value = translations[currentLang];

    for (const k of keys) {
      value = value?.[k];
      if (value === undefined) {
        if (checkStartsWith(key, 'outboundNames.')) {
          return key.split('.')[1];
        }
        return key;
      }
    }
    return value;
  };
}



```

我帮你把 Sublink Worker 的**底层源代码**“解剖”了一遍，终于找到了“罪魁祸首”！

你刚才反复测试依然被覆盖的原因，**绝对不是你的配置语法（比如 DOMAIN-SUFFIX）有问题**，而是 Sublink Worker 程序的**底层硬编码逻辑**导致的。

### 🔍 为什么基础配置里的 rules 永远不生效？

1. **强制屏蔽与删除**：在 Sublink Worker 的源码 (`BaseConfigBuilder.js`) 中，当它读取到你粘贴的 Base Config 时，有一个名为 `blacklistedKeys` 的拦截机制，它会**直接把 `rules` 和 `rule-providers` 这两项彻底删掉并无视**。
2. **强制自动生成**：它只允许通过它网页上的“规则集”或者“自定义规则”来生成路由。
3. **强制注入兜底**：如果在网页上你把所有基础规则（Rulesets）的勾全部取消，程序会触发一个 `length === 0` 的判断，认为你“忘记”选规则了，于是强行往你的配置里塞入一个名为 `minimal`（最小化）的默认规则集（包含 `🔒 国内服务` 和 `🌐 非中国`）。

这也就解释了为什么你在 Base Config 里写得再完美、网页上的勾取消得再干净，最后生成的配置依然会被系统“强暴”覆盖。

---

### 🛠️ 终极网页端解决方案

既然程序的机制是这样，我们就按照它设计的“正确姿势”来喂给它数据。
我们需要分两步走：**在 Base Config 中只保留节点与策略组，在 Custom Rules（自定义规则）中用 JSON 喂给它路由规则**。

#### 第一步：修改你的基础配置 (Base Config)

请把之前那一长串代码里的 `rules:` 部分**完全删掉**，只保留网络配置和分组。请直接复制以下精简后的代码粘贴到“基础配置”框中：

```yaml
port: 7890
socks-port: 7891
allow-lan: false
mode: Rule
log-level: info
dns:
  enable: true
  ipv6: true
  respect-rules: true
  enhanced-mode: fake-ip
  nameserver:
    - https://120.53.53.53/dns-query
    - https://223.5.5.5/dns-query
  proxy-server-nameserver:
    - https://120.53.53.53/dns-query
    - https://223.5.5.5/dns-query

proxy-groups:
  - name: 🚀 节点选择
    type: select
    proxies: [♻️ 自动选择, ⚖️ 负载均衡, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇺🇲 美国节点, 🇰🇷 韩国节点, 📥 下载节点, 🚀 手动切换, 🐢 慢速节点, DIRECT]
  - name: ♻️ 自动选择
    type: url-test
    url: http://cp.cloudflare.com/generate_204
    interval: 600
    tolerance: 50
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文|官网|剩余|到期|hk|港|hongkong)).*'
  - name: ⚖️ 负载均衡
    type: fallback
    url: http://cp.cloudflare.com/generate_204
    interval: 600
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文|hk|港|hongkong)).*'
  - name: 💰 加密货币
    type: select
    proxies: [🇯🇵 日本节点, 🇺🇲 美国节点, 🚀 手动切换]
  - name: 💬 Ai平台
    type: select
    proxies: [🇺🇲 美国节点, 🇯🇵 日本节点, 🐢 慢速节点]
  - name: 🔞 成人内容
    type: select
    proxies: [🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择, 🐢 慢速节点]
  - name: ☁️ 海外网盘
    type: select
    proxies: [📥 下载节点, 🚀 节点选择, 🐢 慢速节点]
  - name: 📺 哔哩哔哩
    type: select
    proxies: [🎯 全球直连, 🇭🇰 香港节点, 📥 下载节点]
  - name: 📹 油管视频
    type: select
    proxies: [♻️ 自动选择, 🚀 节点选择, 🇯🇵 日本节点, 🇭🇰 香港节点, 🇺🇲 美国节点, 📥 下载节点, 🐢 慢速节点]
  - name: 📲 电报消息
    type: select
    proxies: [♻️ 自动选择, 🚀 节点选择, 📥 下载节点, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇺🇲 美国节点, 🐢 慢速节点]
  - name: 🌍 国外媒体
    type: select
    proxies: [🚀 节点选择, 📥 下载节点, ♻️ 自动选择, 🇭🇰 香港节点, 🇯🇵 日本节点, 🇺🇲 美国节点, 🐢 慢速节点]
  - name: 🎮 游戏平台
    type: select
    proxies: [DIRECT, 📥 下载节点, 🐢 慢速节点, 🇯🇵 日本节点, 🚀 节点选择]
  - name: 📢 谷歌FCM
    type: select
    proxies: [♻️ 自动选择, 🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择]
  - name: Ⓜ️ 微软Bing
    type: select
    proxies: [♻️ 自动选择, 🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择]
  - name: Ⓜ️ 微软服务
    type: select
    proxies: [DIRECT, 🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择]
  - name: 🍎 苹果服务
    type: select
    proxies: [DIRECT, 🇺🇲 美国节点, 🇯🇵 日本节点, 🚀 节点选择]
  - name: 🎯 全球直连
    type: select
    proxies: [DIRECT]
  - name: 🚫 广告拦截
    type: select
    proxies: [REJECT, DIRECT]
  - name: 🐟 漏网之鱼
    type: select
    proxies: [⚖️ 负载均衡, ♻️ 自动选择, 🇺🇲 美国节点, 🇯🇵 日本节点, 🇰🇷 韩国节点]
  - name: 📥 下载节点
    type: select
    include-all: true
    filter: '(?i)(wx|s801)'
  - name: 🇭🇰 香港节点
    type: select
    include-all: true
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文)).*(hk|港|hongkong)'
  - name: 🇯🇵 日本节点
    type: select
    include-all: true
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文)).*(jp|日本|tokyo|c93s4)'
  - name: 🇰🇷 韩国节点
    type: select
    include-all: true
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文)).*(kr|韩|seoul)'
  - name: 🇺🇲 美国节点
    type: select
    include-all: true
    filter: '(?i)^(?!.*(jgw|Oracle|甲骨文|c93s4|s801)).*(us|美|unitedstates|c93s|\.top-(?:VLESS_Reality_Vision|singbox_naive|anytls))'
  - name: 🐢 慢速节点
    type: select
    include-all: true
    filter: '(?i)(jgw|Oracle|甲骨文)'
  - name: 🚀 手动切换
    type: select
    include-all: true
    filter: '.*'

```

#### 第二步：配置 Custom Rules (自定义规则)

网页上找到 **【自定义规则 (Custom Rules)】** 这个区域，切换到 **JSON 视图**。
我把你所有精心挑选的“拦截、网盘、成人、电报”规则，全部转化为了 Sublink Worker 识别的 JSON 代码。请把下面这串 JSON **完整地粘贴进去**：

```json
[
  {
    "name": "🚫 广告拦截",
    "site": "category-ads-all"
  },
  {
    "name": "💰 加密货币",
    "site": "category-cryptocurrency"
  },
  {
    "name": "💬 Ai平台",
    "site": "category-ai-!cn"
  },
  {
    "name": "🔞 成人内容",
    "site": "category-porn",
    "domain_keyword": "hanime1,bika"
  },
  {
    "name": "☁️ 海外网盘",
    "site": "dropbox,mega,pikpak,terabox,usersdrive,onedrive,pstorage",
    "domain_suffix": "mediafire.com,pcloud.com,1fichier.com,uptobox.com,gofile.io,catbox.moe,filen.io,sync.com",
    "domain_keyword": "rapidgator"
  },
  {
    "name": "📲 电报消息",
    "ip": "telegram"
  },
  {
    "name": "📹 油管视频",
    "site": "youtube"
  },
  {
    "name": "📺 哔哩哔哩",
    "site": "bilibili"
  },
  {
    "name": "🍎 苹果服务",
    "site": "apple"
  },
  {
    "name": "Ⓜ️ 微软服务",
    "site": "microsoft"
  },
  {
    "name": "🌍 国外媒体",
    "site": "geolocation-!cn"
  },
  {
    "name": "🎯 全球直连",
    "site": "geolocation-cn",
    "ip": "cn"
  }
]

```

#### 第三步：如何勾选生成

1. 填好你的原始节点链接。
2. **【关键】** 在“基础规则 (Rulesets)”区域，**随便打勾保留一个规则**（比如就只保留“🔒 国内服务”），千万不要全部取消！这样它才不会触发那个把规则全部覆盖掉的“最小化防御机制”。
3. 因为我们在自定义 JSON 里设置了很高的优先级，它会自动跑到你的配置文件最前面拦截流量，所以它底部附带的那一点系统规则完全不会影响你，你可以当它们不存在。
4. 点击**“生成订阅”**。

快去试试这套顺应它底层机制的最终解法！成功生成后，你就可以得到一个完美的永久在线更新链接了！
