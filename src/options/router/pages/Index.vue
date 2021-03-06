<template>
    <div class="page">
        <div class="boxed-group">
            <div class="title">LifeScope Browser Extension Options</div>

            <div class="padded paragraphed mobile-margin">
                <p>
                    LifeScope can help you organize and search your time on the web.
                    This can be done by navigating to a site, clicking the LifeScope extension icon in the upper right,
                    and selecting either 'Start tracking this domain' or 'Start tracking this URL'.
                    You can also manually add the domain or URL to the list below.
                </p>
                <p v-if="browserName === 'Chrome' || browserName === 'Firefox'">
                    When you add a domain or URL to the whitelist, all visits to that domain or URL will be indexed from
                    your browser.
                </p>
                <p v-else-if="browserName === 'Microsoft Edge'">
                    Microsoft Edge unfortunately does not let us access your browser history at this time.
                    We therefore cannot retrieve any previous visits to domains or URLs that you have approved, and can
                    only track future visits.
                    Note that you must be logged into LifeScope on that device for future visits to be recorded.
                </p>
                <p>Deleting an item from the list will stop tracking it, but it will <u>not</u> delete visits already
                    recorded for that domain or URL.</p>
                <p>
                    At this time, there is no option for selectively deleting data gathered using this extension, but
                    you can delete all indexed web visits from that browser in the LifeScope app.
                </p>
                <div v-if="$data.connection && $data.connection.enabled === false">
                    <h2>Extension Connection is disabled</h2>

                    <div>
                        <p>The Connection for this browser extension is currently disabled. No new visits to any of the
                            domains or URLs in your list are being recorded.</p>
                        <p>Any new domains or URLs that you add to the list will not be indexed until the Browser
                            Extension Connection is enabled.</p>
                    </div>
                </div>
                <div class="whitelist">
                    <h3>Tracked Domain URL List</h3>
                    <div>
                        <p>Enter web domains or URLs of websites for LifeScope to track, such as 'wikipedia.org' or
                            'https://hulu.com/rick-and-morty'.</p>
                    </div>
                    <form v-on:submit.prevent="addWhitelistEntry">
                        <div class="add-domain flexbox flex-x-center">
                            <div class="text-box">
                                <input type="text"
                                       placeholder="google.com"
                                       v-model="newDomain"
                                >
                            </div>
                            <i class="fa fa-plus"
                               v-on:click="addWhitelistEntry"
                            ></i>
                            <span v-if="$data.invalidDomain === true">That was not a valid domain or site.</span>
                        </div>
                    </form>
                    <div class="entries">
                        <div v-for="entry in sortedEntries"
                             class="entry"
                        >
                            <span>{{ entry }}</span>
                            <i class="delete fa fa-times"
                               v-on:click="deleteWhitelistEntry(entry)"
                            ></i>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
	import Bowser from 'bowser';
	import _ from 'lodash';
	import axios from 'axios';
	import gql from 'graphql-tag';
	import url from 'url';

	let currentBrowser;

	let browser = Bowser.getParser(window.navigator.userAgent);
	let browserName = browser.getBrowserName();

	switch(browserName) {
		case ('Chrome'):
			currentBrowser = chrome;

			break;

		case ('Firefox'):
			currentBrowser = browser;

			break;

		case ('Microsoft Edge'):
			currentBrowser = browser;

			break;

		default:
			currentBrowser = browser;
	}

	let domainRegex = /^([a-zA-Z0-9]+\.)+[a-zA-Z0-9]+$/;
	let siteRegex = /^(http(s)?:\/\/)([a-zA-Z0-9]+\.)+[a-zA-Z0-9]+(\/([.a-zA-Z0-9_-~!$&'()*+,;=:@])+)*\/?/;

	export default {
		data() {
			return {
				domain: null,
				loggedIn: false,
				newDomain: '',
				connection: {},
				invalidDomain: false,
				invalidSite: false
			};
		},

		computed: {
			sortedEntries() {
				return this.$store.state.whitelist.sort();
			},

			browserName: function() {
				return browserName;
			}
		},

		methods: {
			addWhitelistEntry: async function() {
				let self = this;

				await this.$store.dispatch({
					type: 'loadUserSettings'
				});

				this.$data.domain = this.$data.newDomain;

				this.$data.newDomain = '';

				let parsedUrl = url.parse(this.$data.domain);

				let condensedUrl = this.$data.domain;

				if (parsedUrl.protocol && parsedUrl.host && parsedUrl.pathname) {
					condensedUrl = parsedUrl.protocol + '//' + parsedUrl.host;

					if (parsedUrl.pathname !== '/') {
						condensedUrl += parsedUrl.pathname;
					}
				}


				let domainWhitelistExists = this.$store.state.whitelist.indexOf(this.$data.domain);

				let siteWhitelistExists = this.$store.state.whitelist.indexOf(condensedUrl);

				if (this.$data.domain.match(domainRegex) == null && condensedUrl.match(siteRegex) == null) {
					this.$data.invalidDomain = true;

					setTimeout(function() {
						self.$data.invalidDomain = false;
					}, 2000);

					return;
				}

				if (siteWhitelistExists === -1) {
					this.$store.state.whitelist.push(condensedUrl);

					await this.$store.dispatch({
						type: 'saveUserSettings'
					});

					currentBrowser.runtime.sendMessage({
						data: 'triggerHistoryCrawl',
						connection: this.$data.connection
					});

					return;
				}

				if (domainWhitelistExists === -1) {
					this.$store.state.whitelist.push(this.$data.domain);

					this.$store.dispatch({
						type: 'saveUserSettings'
					});

					currentBrowser.runtime.sendMessage({
						data: 'triggerHistoryCrawl',
						connection: this.$data.connection
					});
				}
			},

			deleteWhitelistEntry: function(domain) {
				let index = _.findIndex(this.$store.state.whitelist, function(item) {
					return item === domain;
				});

				if (index >= 0) {
					this.$store.state.whitelist.splice(index, 1);

					this.$store.dispatch({
						type: 'saveUserSettings'
					});
				}
			}
		},

		mounted: async function() {
			let self = this;
			let $apollo = this.$apollo.provider.defaultClient;

			await self.$store.dispatch({
				type: 'loadUserSettings'
			});

			let sessionIdCookie = await new Promise(function(resolve, reject) {
				currentBrowser.cookies.get({
					url: 'https://app.lifescope.io',
					name: 'sessionid'
				}, function(results) {
					resolve(results);
				});
			});

			if (sessionIdCookie != null) {
				let result;

				let csrfResponse = await axios.get('https://api.lifescope.io/csrf');
				self.$store.state.csrf_token = csrfResponse.data ? csrfResponse.data.csrf_token : null;

				try {
					result = await $apollo.query({
						query: gql`query getBrowserConnection($browser: String!) {
						connectionBrowserOne(browser: $browser) {
							id,
							enabled,
							provider_id,
							provider_id_string
						}
					}`,
						variables: {
							browser: browserName
						},
						fetchPolicy: 'no-cache'
					});
				}
				catch (err) {
					self.$data.connection = {};

					return;
				}

				let existingBrowserConnection = result.data.connectionBrowserOne;

				if (existingBrowserConnection == null) {
					result = await $apollo.mutate({
						mutation: gql`mutation createBrowserConnection($browser: String!) {
                                connectionCreateBrowser(browser: $browser) {
                                    id,
                                    enabled,
									provider_id,
									provider_id_string
                                }
                            }`,
						variables: {
							browser: browserName
						},
						fetchPolicy: 'no-cache'
					});

					existingBrowserConnection = result.data.connectionCreateBrowser;

					self.$store.state.whitelist = [];
					self.$store.state.whitelistPending = {};
					self.$store.state.whitelistHistory = [];

					self.$store.dispatch({
						type: 'saveUserSettings'
					});
				}

				self.$data.connection = existingBrowserConnection || {};
			}
			else {
				self.$data.connection = {};
			}
		}
	};
</script>
