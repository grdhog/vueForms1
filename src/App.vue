<script>
import { defineComponent } from 'vue'
import HelloWorld from './components/HelloWorld.vue'

import { response } from './sample.js'


export default defineComponent({
  name: 'App',
  components: {
    HelloWorld,
  },
  data: {
    form: {},
    state: 'entering',
    displayModal: true,
    response: {},
    allowInvalidSubmission: false,
    apiURL: 'http://54.252.3.209:8000/test-form-submission/128/',
    recaptchaID: '',
  },
  beforeMount: function () {
    console.log('beforeMount bro!')
    var oReq = new XMLHttpRequest()
    var that = this
    oReq.addEventListener('load', function () {
      var json = JSON.parse(this.responseText)
      that.form = json.form;
    });
    oReq.addEventListener('error', function() {
      console.log('Test server not available using static copy of JSON');
      that.form = response.form;
    });
    oReq.open('GET', this.apiURL)
    oReq.send()
  },
  updated: function () {
    if (this.recaptchaID === '') this.createCaptcha()
  },
  methods: {
    showCaptcha: function () {
      alert(grecaptcha.getResponse(this.recaptchaID))
    },
    correctCaptcha: function () {
      this.$refs.submit.disabled = false
    },
    expiredCaptcha: function () {
      this.$refs.submit.disabled = true
    },
    createCaptcha: function () {
      console.log('the dom is fully rendered now')
      this.recaptchaID = grecaptcha.render(this.$refs.recaptcha, {
        sitekey: '6LeIxAcTAAAAAJcZVRqyHh71UMIEGNQ_MXjiZKhI',
        callback: this.correctCaptcha,
        'expired-callback': this.expiredCaptcha,
      })
    },
    getCookie: function (name) {
      var match = document.cookie.match(new RegExp('(^| )' + name + '=([^;]+)'))
      if (match) return match[2]
    },
    submit: function () {
      console.log('Submitting to URL ' + this.apiURL)
      var that = this
      var XHR = new XMLHttpRequest()
      var fdata = new FormData()
      // add recaptcha
      fdata.append('g-recaptcha-response', grecaptcha.getResponse(this.recaptchaID))

      for (const field of that.form.fields) {
        if (field.field_type == 'checkboxes') {
          field.value.forEach((val) => fdata.append(field.name, val))
        } else {
          fdata.append(field.name, field.value)
        }
      }
      for (const pair of fdata.entries()) {
        console.log(`${pair[0]}, ${pair[1]}`)
      }

      XHR.addEventListener('load', function () {
        console.log('this.responseText', this.responseText)
        //this is the loads this
        let json = JSON.parse(this.responseText)
        that.response = json
      })
      XHR.addEventListener('error', function () {
        alert('Oops! Something went wrong.')
      })
      XHR.open('POST', this.apiURL)
      XHR.send(fdata)
    },
    handleBlur: function (event) {
      //console.log('handleBlur', event.target.name)
      if (!this.getField(event.target.name).touched) this.getField(event.target.name).touched = true
      this.validateAll()
    },
    getField: function (name) {
      for (const field of this.form.fields) {
        if (field.name === name) return field
      }
    },
    validateAll: function () {
      //console.log('validateAll')
      for (const field of this.form.fields) {
        if (field.touched) this.validateField(field)
      }
    },
    validateField: function (field) {
      field.error = ''
      //console.log('validateField', field.name, field.value, typeof field.value)

      // Handle required
      if (field.required) {
        let errorText = this.required(field)
        if (errorText !== '') {
          field.error = errorText
          return
        }
      }

      for (const func of field.validation) {
        //console.log('running', func, 'on', field.name)
        let errorText = this[func](field)
        if (errorText !== '') {
          field.error = errorText
          break
        }
      }
    },
    errorCount: function () {
      let numErrors = 0
      this.form.fields.forEach((field) => {
        if (field.error !== '') numErrors++
      })
      return numErrors
    },
    setAllTouched: function () {
      //console.log('setAllTouched')
      for (const field of this.form.fields) {
        if (!field.touched) field.touched = true
      }
    },
    attemptSubmit: function (event) {
      event.preventDefault()
      this.setAllTouched()
      this.validateAll()
      if (this.allowInvalidSubmission || this.errorCount() === 0) {
        this.submit()
      }
    },
    showModal: function (msg, interval) {
      this.displayModal = true
      this.$refs.modal.style.display = 'block'
      //this.modalMessage = msg
      if (interval) setTimeout(this.closeModal, interval)
    },
    closeModal: function () {
      this.displayModal = false
      this.$refs.modal.style.display = 'none'
    },
    required: function (field) {
      if (typeof field.value === 'boolean') {
        return field.value ? '' : 'Required'
      } else if (Array.isArray(field.value)) {
        //console.log('value.length:', field.value.length)
        return field.value.length !== 0 ? '' : 'Required'
      } else {
        return field.value.trim() !== '' ? '' : 'Required'
      }
    },
    email: function (field) {
      return /^\w+([\.-]?\w+)*@\w+([\.-]?\w+)*(\.\w{2,3})+$/.test(field.value) ? '' : 'Invalid email'
    },
  },
})
</script>

<template>
  <div id="app">
    <form method="POST">
      <h2>Test form</h2>
      <input type="hidden" name="csrfmiddlewaretoken" :value="form.csrf_token" />
      <template v-for="(field, i) in form.fields">
        <template v-if="['singleline', 'email', 'number', 'url'].indexOf(field.field_type) !== -1">
          <div class="form-group" :class="{ 'has-error': field.error }">
            <label class="control-label required" :for="'id_' + field.name">{{ field.label }}</label>
            <input
              :type="field.field_type"
              class="form-control"
              maxlength="255"
              :name="field.name"
              :id="'id_' + field.name"
              v-on:blur="handleBlur"
              v-model="field.value"
            />
            <span class="help-block">touched: {{ field.touched }}</span>
          </div>
        </template>

        <template v-else-if="field.field_type == 'dropdown'">
          <div class="form-group" :class="{ 'has-error': field.error }">
            <label class="control-label required">{{ field.label }}</label>
            <select
              class="form-control"
              :id="'id_' + field.name"
              v-model="field.value"
              v-on:blur="handleBlur"
              :name="field.name"
            >
              <option v-for="(option, i) in field.options" class="checkbox">{{ option }}</option>
            </select>
            <span class="help-block">touched: {{ field.touched }}</span>
          </div>
        </template>

        <template v-else-if="field.field_type == 'multiline'">
          <div class="form-group" :class="{ 'has-error': field.error }">
            <label class="control-label required" for="id_textarea1">{{ field.label }}</label>
            <textarea
              class="form-control"
              v-model="field.value"
              v-on:blur="handleBlur"
              cols="40"
              rows="4"
              :name="field.name"
              :id="'id_' + field.name"
            ></textarea>
            <span class="help-block">touched: {{ field.touched }}</span>
          </div>
        </template>

        <template v-else-if="field.field_type == 'checkbox'">
          <div class="form-group" :class="{ 'has-error': field.error }">
            <div class="checkbox">
              <label class="control-label single required"
                ><input
                  type="checkbox"
                  v-model="field.value"
                  :name="field.name"
                  :id="'id_' + field.name"
                  v-on:blur="handleBlur"
                />{{ field.label }}</label
              >
            </div>
            <span class="help-block">touched: {{ field.touched }}</span>
          </div>
        </template>

        <template v-else-if="field.field_type == 'checkboxes'">
          <div class="form-group" :class="{ 'has-error': field.error }">
            <label class="control-label required">{{ field.label }}</label>
            <div v-for="(option, i) in field.options" class="checkbox">
              <label>
                <input
                  type="checkbox"
                  v-model="field.value"
                  v-on:blur="handleBlur"
                  :name="field.name"
                  :id="'id_' + field.name + '_' + i"
                  :value="option"
                />{{ option }}</label
              >
            </div>
            <span>touched: {{ field.touched }}</span>
          </div>
        </template>

        <template v-else-if="field.field_type == 'radio'">
          <div class="form-group" :class="{ 'has-error': field.error }">
            <label class="control-label required">Radios</label>
            <div v-for="(option, i) in field.options" class="radio">
              <label>
                <input
                  type="radio"
                  v-model="field.value"
                  v-on:blur="handleBlur"
                  :name="field.name"
                  :id="'id_' + field.name + '_' + i"
                  :value="option"
                />
                {{ option }}
              </label>
            </div>
            <span>touched: {{ field.touched }}</span>
          </div>
        </template>

        <template v-else-if="field.field_type === 'wax'">
          <input
            :type="field.field_type"
            class="wax"
            maxlength="255"
            :name="field.name"
            :id="'id_' + field.name"
            v-on:blur="handleBlur"
            v-model="field.value"
          />
        </template>
      </template>
      <div ref="recaptcha"></div>

      <button disabled type="submit" ref="submit" v-on:click="attemptSubmit" class="btn btn-default">Submit</button>
    </form>
    <div
      ref="modal"
      class="modal fade"
      :class="{ in: displayModal }"
      id="vueFormModal"
      tabindex="-1"
      role="dialog"
      aria-labelledby="vueFormModalLabel"
    >
      <div class="modal-dialog" role="document">
        <div class="modal-content">
          <div class="modal-body">Submitting please wait...</div>
          <div class="modal-footer">
            <button type="button" class="btn btn-default" v-on:click="closeModal">Close</button>
          </div>
        </div>
      </div>
    </div>

    <p>{{ form }}</p>
    <p>Response:</p>
    <p>{{ response }}</p>
  </div>
</template>

<style>
.wax {
  padding: 0;
  margin: 0;
  width: 1px;
  height: 1px;
  border: 0;
  display: block;
}

form {
  max-width: 600px;
  margin: auto;
}
.checkbox .single {
  font-weight: 700;
}
label.required:after {
  color: red;
  content: '  *';
}
</style>
