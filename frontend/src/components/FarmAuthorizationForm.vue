<template>
    <form>
        <div class="subtitle-1 text--primary">
            Authorize your farmOS server with the {{appName}}. This will give the {{appName}} access to your data
            at the level you define.
        </div>
        <br>
        <div class="headline text--primary">
            Farm Info
        </div>
        <div class="text--primary">
            Verify that the following information is correct:
        </div>
        <v-text-field label="Farm Name" v-model="farmName" readonly></v-text-field>
        <v-text-field label="URL" v-model="farmUrl" readonly></v-text-field>


        <div class="headline text--primary">
            Permissions
        </div>
        <v-checkbox
                v-model="oauthScopes"
                label="farmOS API Access"
                hint="Allow access to the entire farmOS API"
                value="farmos_restws_access"
                persistent-hint
                disabled
        ></v-checkbox>
    </form>
</template>

<script lang="ts">
    import { Component, Vue, Prop } from 'vue-property-decorator';
    import {FarmProfileAuthorize} from '@/interfaces';
    import {dispatchAuthorizeFarm} from '@/store/farm/actions';

    @Component
    export default class FarmAuthorizationForm extends Vue {
        @Prop({default: false}) public appName!: string;
        @Prop({default: false}) public farmUrl!: string;
        @Prop({default: false}) public farmName!: string;
        @Prop({default: false}) public farmId!: number;
        @Prop({default: false}) public apiToken!: string;

        // Enable the farmos_restws_access scope by default.
        public oauthScopes: string[] = ['farmos_restws_access'];

        // Generate a random string for the state param of OAuth Authorization Flow.
        public authState: string =
            Math.random().toString(36).substring(2, 15) + Math.random().toString(36).substring(2, 15);

        // Reference to popup window.
        public windowObjectReference: any = null;

        public openSignInWindow() {
            // Emit event to update authstatus.
            this.$emit('update:authstatus', 'started');

            const windowFeatures = 'toolbar=no, menubar=no, width=600, height=700, top=100, left=100';

            // Build the OAuth query parameters.
            const responseType = 'code';
            const clientID = 'farmos_api_client';
            const scopes = this.cleanOAuthStrings();
            const redirectURI = `${this.farmUrl}/api/authorized`;
            const state = this.authState;
            const queryParams = `?response_type=${responseType}&client_id=${clientID}&scope=${scopes}&redirect_uri=${redirectURI}&state=${state}`;

            const oauthPath = '/oauth2/authorize';

            // Open a pop up window with the OAuth Authorization URL.
            if (this.windowObjectReference === null || this.windowObjectReference.closed) {
                this.windowObjectReference =
                    window.open(this.farmUrl + oauthPath + queryParams, 'farmOS Login', windowFeatures);
                this.windowObjectReference.focus();
            } else {
                this.windowObjectReference.focus();
            }

            // Add listener to retrieve the OAuth Code
            window.addEventListener('message', (event) => this.receiveMessage(event), false);
        }

        public async receiveMessage(event) {
            // Make sure the message came from the farmOS server.
            if (event.origin !== this.farmUrl) {
                return;
            }

            // Build a URLSearchParams from the message data.
            const params = new URLSearchParams(event.data.substr(1));

            // Parse out the `code` and `state` values.
            const code = params.get('code') as string;
            const state = params.get('state') as string;

            // Make sure the state did not change.
            if (state !== this.authState) {
                return null;
            }

            // Build the payload for the backend to request a token.
            const authValues: FarmProfileAuthorize = {
                grant_type: 'authorization_code',
                code,
                state,
                client_id: 'farmos_api_client',
                client_secret: 'client_secret',
            };

            // Dispatch API call to backend.
            await dispatchAuthorizeFarm(this.$store, { id: this.farmId, authValues, apiToken: this.apiToken });

            // Emit events to update authorization status.
            this.$emit('update:authstatus', 'completed');
            this.$emit('authorizationcomplete');
        }

        public cleanOAuthStrings() {
            // Change the OAuth Scopes from a list of strings to one space separated string of scopes.
            // These are embedded in the query parameters.
            let allScopes: string = '';
            for (const scope of this.oauthScopes) {
                allScopes += scope + ' ';
            }
            return allScopes;
        }

    }


</script>

<style scoped>

</style>