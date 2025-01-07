<!-- eslint-disable vue/valid-v-slot -->
<template>
  <v-container>
    <v-stepper :items="['Upload config','Edit keybinds','Save new config']" v-model="step" hide-actions>
      <template v-slot:item.1>
        <v-card>
          <v-card-text>
            Upload your xemu.toml file. On Windows, it can be found at <code> %USERPROFILE%\AppData\Roaming\xemu\xemu\xemu.toml </code> or in the same folder as the Xemu executable
          </v-card-text>
          <v-card-text>
            <v-row>
              <v-col cols="9">
                <v-file-input label="xemu.toml" accept=".toml" v-model="file"/>
              </v-col>
              <v-col cols="3">
                <v-btn @click="parse_toml" color="primary">
                  Upload
                </v-btn>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>
      </template>
      <template v-slot:item.2>
        <v-card>
          <v-card-title> Modify keybinds here </v-card-title>
          <v-row>
            <template v-for="(key,i) of keys" :key="`row-${i}`" >
              <v-col cols="6" class="text-right">
                {{  key.name }}: {{ data.input.keyboard_controller_scancode_map[key.key] ? (sdl_to_key(data.input.keyboard_controller_scancode_map[key.key]) || {name: 'Unknown'}).name : 'Unbound' }}
              </v-col>
              <v-col cols="6" >
                <v-btn @click="begin_rebind(key)" color="primary"> Rebind {{ key.name }} </v-btn>
              </v-col>
            </template>
          </v-row>
          <v-card-actions>
            <v-btn @click="this.step = 1"> Back </v-btn>
            <v-btn @click="this.step = 3"> Next </v-btn>
          </v-card-actions>
        </v-card>
      </template>
      <template v-slot:item.3>
        <v-card>
          <v-card-title> Download config file </v-card-title>
          <v-card-text>
            <v-btn prepend-icon="mdi-download" @click="save_toml"> Download config file </v-btn>
          </v-card-text>
          <v-card-actions>
            <v-btn @click="this.step = 2"> Back </v-btn>
          </v-card-actions>
        </v-card>
      </template>
    </v-stepper>
    <v-dialog max-width="600" v-model="show_input_dialog">
      <v-card @keypress="rebind_key">
        <v-card-title>
          Press the key you want to bind to {{ awaited_key.name }}
        </v-card-title>
        <v-card-text>
          Key: {{  awaited_key.caught.name || 'Unbound' }}
        </v-card-text>
        <v-card-actions>
          <v-btn @click="save_key" color="primary"> Save </v-btn>
          <v-btn @click="cancel_key" color="danger"> Cancel </v-btn>
          <v-btn @click="unbind_key" color="danger"> Unbind </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <a ref="download" download="xemu.toml"></a>
  </v-container>
</template>

<script>
import keys from '@/assets/keys.json';
import lut from '@/assets/keys-lut.json';

import * as toml from 'smol-toml';
  export default {
    name: 'Home',
    data: () => ({
        file: null,
        step: null,
        data: null,
        keys: [...keys],
        show_input_dialog: false,
        awaited_key: {
          name: '',
          key: '',
          caught: {
            keycode: 0,
            sdl: 0,
            name: '',
          }
        },
    }),
    mounted() {
      document.addEventListener('keyup',this.rebind_key);
    },
    unmounted(){
      document.removeEventListener('keyup',this.rebind_key);
    },
    methods: {
      async parse_toml(){
        if (!this.file) { return; }
        const text = await this.file[0].text();
        const data = toml.parse(text);
        if (data.input == undefined) {
          data.input = {};
        }
        if (data.input.keyboard_controller_scancode_map == undefined) {
          data.input.keyboard_controller_scancode_map = {};
        }
        // store config for now
        this.data = data;
        this.step = 2;
      },
      begin_rebind(key) {
        this.awaited_key.name = key.name;
        this.awaited_key.key = key.key;
        this.show_input_dialog = true;
        console.log(key);
        const keydata = this.sdl_to_key(this.data.input.keyboard_controller_scancode_map[key.key]);
        console.log(keydata);
        if (keydata != undefined) {
          this.awaited_key.caught.keycode = keydata.keycode;
          this.awaited_key.caught.sdl = keydata.sdl;
          this.awaited_key.caught.name = keydata.name;
        } else {
          this.awaited_key.caught.keycode = 0;
          this.awaited_key.caught.sdl = 0;
          this.awaited_key.caught.name = '';
        }

      },
      rebind_key(e) {
        if (this.show_input_dialog) {
          console.log(e);
          const key = this.keycode_to_key(e.keyCode);
          this.awaited_key.caught.keycode = key.keycode;
          this.awaited_key.caught.name = key.name;
          this.awaited_key.caught.sdl = key.sdl;
          

        }
      },
      save_key() {
        this.data.input.keyboard_controller_scancode_map[this.awaited_key.key] = this.awaited_key.caught.sdl;
        this.show_input_dialog = false;
      },
      cancel_key() {
        this.show_input_dialog = false;
      },
      unbind_key() {
        delete this.data.input.keyboard_controller_scancode_map[this.awaited_key.key]
        this.show_input_dialog = false;
      },
      keycode_to_key(keycode){
        const key = lut.find(v=>v.keycode == keycode);
        return key;
      },
      sdl_to_key(sdl) {
        const key = lut.find(v=>v.sdl == sdl);
        return key;
      },
      save_toml(){
        const file = toml.stringify(this.data);
        const blob = new Blob([file]);
        const url = URL.createObjectURL(blob);

        console.log(this.$refs.download);
        this.$refs.download.href = url;
        this.$refs.download.click();
      }
    },
}
</script>
