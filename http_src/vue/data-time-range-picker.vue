<!-- (C) 2022 - ntop.org     -->
<template>
<div class="input-group">    
  <div class="form-group">
    <div class="controls d-flex flex-wrap">
      <div class="btn-group me-auto btn-group-sm">
        <slot name="begin"></slot>
        <select-search v-model:selected_option="selected_time_option"
          :id="'time_preset_range_picker'"
          :options="time_preset_list"
          @select_option="change_select_time(null)">
        </select-search>
        <div class="btn-group ms-2">
            <span class="input-group-text">
                <i class="fas fa-calendar-alt"></i>
            </span>
            <input  class="flatpickr flatpickr-input" type="text" placeholder="Choose a date.." data-id="datetime" ref="begin-date">
            <!-- <input ref="begin-date" @change="enable_apply=true" @change="change_begin_date" type="date" class="date_time_input begin-timepicker form-control border-right-0 fix-safari-input"> -->
            <!-- <input ref="begin-time" @change="enable_apply=true" type="time" class="date_time_input begin-timepicker form-control border-right-0 fix-safari-input"> -->
            <span class="input-group-text">
                <i class="fas fa-long-arrow-alt-right"></i>
            </span>
            <input  class="flatpickr flatpickr-input" type="text" placeholder="Choose a date.." data-id="datetime" ref="end-date">
            <!-- <input ref="end-date" @change="enable_apply=true" type="date" class="date_time_input end-timepicker form-control border-left-0 fix-safari-input" style="width: 2.5rem;"> -->
            <!-- <input ref="end-time" @change="enable_apply=true" type="time" class="date_time_input end-timepicker form-control border-left-0 fix-safari-input"> -->
            <span v-show="wrong_date" :title="i18n('wrong_date_range')" style="margin-left:0.2rem;color:red;">
                <i class="fas fa-exclamation-circle"></i>
            </span>
        </div>

        <div class="d-flex align-items-center ms-2">
            <button :disabled="!enable_apply || wrong_date" @click="apply" class="btn btn-sm btn-primary">{{i18n('apply')}}</button>
                
            <div class="btn-group">
                <button @click="jump_time_back()" class="btn btn-sm btn-link" ref="btn-jump-time-back" :title="i18n('date_time_range_picker.btn_move_left')">
                <i class="fas fa-long-arrow-alt-left"></i>
                </button>
                <button @click="jump_time_ahead()" class="btn btn-sm btn-link me-2" ref="btn-jump-time-ahead" :title="i18n('date_time_range_picker.btn_move_right')">
                <i class="fas fa-long-arrow-alt-right"></i>
                </button>
                <button @click="zoom(2)" class="btn btn-sm btn-link" ref="btn-zoom-in" :title="i18n('date_time_range_picker.btn_zoom_in')">
                <i class="fas fa-search-plus"></i>
                </button>
                <button @click="zoom(0.5)" class="btn btn-sm btn-link" ref="btn-zoom-out" :title="i18n('date_time_range_picker.btn_zoom_out')">
                <i class="fas fa-search-minus"></i>
                </button>
                <button :disabled="history_last_status == null" @click="apply_status_by_history()" class="btn btn-sm btn-link" :title="i18n('date_time_range_picker.btn_undo')">
                <i class="fas fa-undo"></i>
                </button>
                <button :disabled="select_time_value == 'custom'" @click="change_select_time()" class="btn btn-sm btn-link" :title="i18n('date_time_range_picker.btn_refresh')">
                <i class="fas fa-sync"></i>
                </button>
		<slot name="extra_buttons"></slot>
            </div>
        </div>
      </div>
    </div>
  </div>  
</div>
</template>

<script>
import { default as SelectSearch } from "./select-search.vue";

export default {
    components: {
	'select-search': SelectSearch,
    },
    props: {
	id: String,
	enable_refresh: Boolean,
    },
    watch: {
	"enable_refresh": function(val, oldVal) {
	    if (val == true) {
		this.start_refresh();
	    } else if (this.refresh_interval != null) {
		clearInterval(this.refresh_interval);
		this.refresh_interval = null;
	    }
	}
    },	
    emits: ["epoch_change"],
    /** This method is the first method of the component called, it's called before html template creation. */
    created() {	
    },
    /** This method is the first method called after html template creation. */
    mounted() {
	let epoch_begin = ntopng_url_manager.get_url_entry("epoch_begin");
	let epoch_end = ntopng_url_manager.get_url_entry("epoch_end");
	if (epoch_begin != null && epoch_end != null) {
	    // update the status
	    
            ntopng_events_manager.emit_event(ntopng_events.EPOCH_CHANGE, { epoch_begin: Number.parseInt(epoch_begin), epoch_end: Number.parseInt(epoch_end) }, this.$props.id);
	}
	let me = this;
	let f_set_picker = (picker, var_name) => {
	    return flatpickr($(this.$refs[picker]), {
		enableTime: true,
		dateFormat: "d/m/Y H:i",
		//altInput: true,
		//dateFormat: "YYYY-MM-DD HH:mm",
		//altFormat: "d-m-Y H:i",
		//locale: "it",
		time_24hr: true,
		clickOpens: true,		
		//mode: "range",
		//static: true,
		onChange: function(selectedDates, dateStr, instance) {
		    me.enable_apply = true;
		    me.wrong_date = me.flat_begin_date.selectedDates[0].getTime() > me.flat_end_date.selectedDates[0].getTime();
		    //me.a[data] = d;
		},
	    });
	};
	this.flat_begin_date = f_set_picker("begin-date", "begin_date");
	this.flat_end_date = f_set_picker("end-date", "end_date");
        ntopng_events_manager.on_event_change(this.$props.id, ntopng_events.EPOCH_CHANGE, (new_status) => this.on_status_updated(new_status), true);
	
	// notifies that component is ready
	//console.log(this.$props["id"]);
	ntopng_sync.ready(this.$props["id"]);
	if (this.enable_refresh) {
	    this.start_refresh();
	}
    },
    
    /** Methods of the component. */
    methods: {
	start_refresh: function() {
	    this.refresh_interval = setInterval(() => {
		let value = this.selected_time_option?.value;
		if (this.enable_refresh && value != null && value != "custom") {
		    this.update_from_interval = true;
		    this.change_select_time(true);
		}
	    }, this.refresh_interval_seconds * 1000);
	    // }, 10* 1000);
	},
	utc_s_to_server_date: function(utc_seconds) {
	    let utc = utc_seconds * 1000;
	    let d_local = new Date(utc);
	    let local_offset = d_local.getTimezoneOffset();
	    let server_offset = moment.tz(utc, ntop_zoneinfo)._offset;
	    let offset_minutes =  server_offset + local_offset;
	    let offset_ms = offset_minutes * 1000 * 60;
	    var d_server = new Date(utc + offset_ms);
	    return d_server;
	},
	server_date_to_date: function(date, format) {
	    let utc = date.getTime();
	    let local_offset = date.getTimezoneOffset();
	    let server_offset = moment.tz(utc, ntop_zoneinfo)._offset;
	    let offset_minutes =  server_offset + local_offset;
	    let offset_ms = offset_minutes * 1000 * 60;
	    var d_local = new Date(utc - offset_ms);
	    return d_local;
	},
        on_status_updated: function(status) {
            let end_date_time_utc = Date.now();        
            // default begin date time now - 30 minutes
            let begin_date_time_utc = end_date_time_utc - 30 * 60 * 1000;
            if (status.epoch_end != null && status.epoch_begin != null
		&& Number.parseInt(status.epoch_end) > Number.parseInt(status.epoch_begin)) {
		status.epoch_begin = Number.parseInt(status.epoch_begin);
		status.epoch_end = Number.parseInt(status.epoch_end);
                end_date_time_utc = status.epoch_end * 1000;
                begin_date_time_utc = status.epoch_begin * 1000;
            } else {
                status.epoch_end = this.get_utc_seconds(end_date_time_utc);
                status.epoch_begin = this.get_utc_seconds(begin_date_time_utc);
		ntopng_url_manager.add_obj_to_url(status);
                this.emit_epoch_change(status, this.$props.id);
            }
	    // this.flat_begin_date.setDate(new Date(status.epoch_begin * 1000));
	    // this.flat_end_date.setDate(new Date(status.epoch_end * 1000));
	    this.flat_begin_date.setDate(this.utc_s_to_server_date(status.epoch_begin));
	    this.flat_end_date.setDate(this.utc_s_to_server_date(status.epoch_end));
            // this.set_date_time("begin-date", begin_date_time_utc, false);
            // this.set_date_time("begin-time", begin_date_time_utc, true);
            // this.set_date_time("end-date", end_date_time_utc, false);
            // this.set_date_time("end-time", end_date_time_utc, true);
            this.set_select_time_value(begin_date_time_utc, end_date_time_utc);
            this.epoch_status = { epoch_begin: status.epoch_begin, epoch_end: status.epoch_end };
	    if (this.update_from_interval == false) {
		this.add_status_in_history(this.epoch_status);
	    }
            this.enable_apply = false;
	    this.update_from_interval = false;
	    ntopng_url_manager.add_obj_to_url(this.epoch_status);
        },
        set_select_time_value: function(begin_utc, end_utc) {
            let s_values = this.get_select_values();
            const tolerance = 60;
            const now = this.get_utc_seconds(Date.now());
            const end_utc_s = this.get_utc_seconds(end_utc);
            const begin_utc_s = this.get_utc_seconds(begin_utc);

	    
            if (this.is_between(end_utc_s, now, tolerance)) {
                if (this.is_between(begin_utc_s, now - s_values.min_5, tolerance)) {
                    this.select_time_value = "min_5";
                } else if (this.is_between(begin_utc_s, now - s_values.min_30, tolerance)) {
                    this.select_time_value = "min_30";
                } else if (this.is_between(begin_utc_s, now - s_values.hour, tolerance)) {
                    this.select_time_value = "hour";
                } else if (this.is_between(begin_utc_s, now - s_values.day, tolerance)) {
                    this.select_time_value = "day";
                } else if (this.is_between(begin_utc_s, now - s_values.week, tolerance)) {
                    this.select_time_value = "week";
                } else if (this.is_between(begin_utc_s, now - s_values.month, tolerance)) {
                    this.select_time_value = "month";
                } else if (this.is_between(begin_utc_s, now - s_values.year, tolerance)) {
                    this.select_time_value = "year";
                } else {
                    this.select_time_value = "custom";
                }
            } else {
                this.select_time_value = "custom";
            }
            
            this.time_preset_list.forEach(element => {
              element.currently_active = false
              if(element.value == this.select_time_value) {
                this.selected_time_option = element;
                element.currently_active = true;
              }
            });
        },
        apply: function() {
            // let date_begin = this.$refs["begin-date"].valueAsDate;
            // let d_time_begin = this.$refs["begin-time"].valueAsDate;
            // date_begin.setHours(d_time_begin.getHours());
            // date_begin.setMinutes(d_time_begin.getMinutes() + d_time_begin.getTimezoneOffset());
            // date_begin.setSeconds(d_time_begin.getSeconds());
            
            // let date_end = this.$refs["end-date"].valueAsDate;
            // let d_time_end = this.$refs["end-time"].valueAsDate;
            // date_end.setHours(d_time_end.getHours());
            // date_end.setMinutes(d_time_end.getMinutes() + d_time_end.getTimezoneOffset());
            // date_end.setSeconds(d_time_end.getSeconds());
            // let epoch_begin = this.get_utc_seconds(date_begin.valueOf());
            // let epoch_end = this.get_utc_seconds(date_end.valueOf());
	    let now_s = this.get_utc_seconds(Date.now());
	    let begin_date = this.server_date_to_date(this.flat_begin_date.selectedDates[0]);
	    let epoch_begin = this.get_utc_seconds(begin_date.getTime());
	    let end_date = this.server_date_to_date(this.flat_end_date.selectedDates[0]);
	    let epoch_end = this.get_utc_seconds(end_date.getTime());
	    if (epoch_end > now_s) {
		epoch_end = now_s;
	    }
            let status = { epoch_begin , epoch_end };
            this.emit_epoch_change(status);
        },
        // set_date_time: function(ref_name, utc_ts, is_time) {
        //     utc_ts = this.get_utc_seconds(utc_ts) * 1000;        
        //     let date_time = new Date(utc_ts);
        //     date_time.setMinutes(date_time.getMinutes() - date_time.getTimezoneOffset());
	//     if (is_time) {
	// 	this.$refs[ref_name].value = date_time.toISOString().substring(11,16);
	//     } else {
	// 	this.$refs[ref_name].value = date_time.toISOString().substring(0,10);
	//     }
        // },
        change_select_time: function(refresh_data) {
            let s_values = this.get_select_values();
            let interval_s = s_values[this.selected_time_option.value];
            let epoch_end = this.get_utc_seconds(Date.now());
            let epoch_begin = epoch_end - interval_s;
            let status = { epoch_begin: epoch_begin, epoch_end: epoch_end, refresh_data };
            this.emit_epoch_change(status);
        },
        get_select_values: function() {
            let min = 60;
            return {
                min_5: min * 5,
                min_30: min * 30,
                hour: min * 60,
                day: this.get_last_day_seconds(), 
                week: this.get_last_week_seconds(), 
                month: this.get_last_month_seconds(), 
                year: this.get_last_year_seconds(),
            };
        },
        get_utc_seconds: function(utc_ts) {
            return Number.parseInt(utc_ts / 1000);
        },
        is_between: function(x, y, tolerance) {
            return x >= y - tolerance && x <= y;
        },
        get_last_day_seconds: function() {
            let t = new Date();
            return this.get_utc_seconds(Date.now() - t.setDate(t.getDate() - 1));
        },
        get_last_week_seconds: function() {
            let t = new Date();
            return this.get_utc_seconds(Date.now() - t.setDate(t.getDate() - 7));
        },
        get_last_month_seconds: function() {
            let t = new Date();
            return this.get_utc_seconds(Date.now() - t.setMonth(t.getMonth() - 1));
        },
        get_last_year_seconds: function() {
            let t = new Date();
            return this.get_utc_seconds(Date.now() - t.setMonth(t.getMonth() - 12));
        },
        zoom: function(scale) {
            if (this.epoch_status == null) { return; }
            let interval = (this.epoch_status.epoch_end - this.epoch_status.epoch_begin) / scale;
            let center = (this.epoch_status.epoch_end / 2 + this.epoch_status.epoch_begin / 2);
            this.epoch_status.epoch_begin = center - interval / 2;
            this.epoch_status.epoch_end = center + interval / 2;
            let now = this.get_utc_seconds(Date.now());
            if (this.epoch_status.epoch_end > now) {
                this.epoch_status.epoch_end = now;
            }
            this.epoch_status.epoch_end = Number.parseInt(this.epoch_status.epoch_end);
            this.epoch_status.epoch_begin = Number.parseInt(this.epoch_status.epoch_begin);
            if (this.epoch_status.epoch_begin == this.epoch_status.epoch_end) {
                this.epoch_status.epoch_begin -= 2;
            }
            this.emit_epoch_change(this.epoch_status);
        },
        jump_time_back: function() {
            if (this.epoch_status == null) { return; }
            const min = 60;
            this.epoch_status.epoch_begin -= (30 * min);
            this.epoch_status.epoch_end -= (30 * min);
            this.emit_epoch_change(this.epoch_status);
        },
        jump_time_ahead: function() {
            if (this.epoch_status == null) { return; }
            const min = 60;
            let previous_end = this.epoch_status.epoch_end;
            let now = this.get_utc_seconds(Date.now());
            
            this.epoch_status.epoch_end += (30 * min);
            if (this.epoch_status.epoch_end > now) {
                this.epoch_status.epoch_end = now;
            }
            this.epoch_status.epoch_begin += (this.epoch_status.epoch_end - previous_end);
            this.emit_epoch_change(this.epoch_status);
        },
        emit_epoch_change: function(epoch_status, id) {
            if (epoch_status.epoch_end == null || epoch_status.epoch_begin == null) { return; };
            this.wrong_date = false;
            if (epoch_status.epoch_begin > epoch_status.epoch_end) {
                this.wrong_date = true;
		return;
            }
	    if (id != this.id) {
		this.on_status_updated(epoch_status);
	    }
            ntopng_events_manager.emit_event(ntopng_events.EPOCH_CHANGE, epoch_status, this.id);
            this.$emit("epoch_change", epoch_status);
        },
	add_status_in_history: function(epoch_status) {
	    this.history_last_status = this.history[this.history.length - 1];
	    if (this.history.length > 5) {
		this.history.shift();
	    }
	    this.history.push(epoch_status);
	},
	
	apply_status_by_history: function() {
	    if (this.history_last_status == null) { return; }
	    this.history.pop();
	    this.history.pop();
	    this.emit_epoch_change(this.history_last_status);
	},
    },
    /**
       Private date of vue component.
    */
  data() {
      return {
    i18n: (t) => i18n(t),
      //status_id: "data-time-range-picker" + this.$props.id,
	  epoch_status: null,
	  refresh_interval: null,
	  refresh_interval_seconds: 60,
	  update_from_interval: false,
	  history: [],
	  history_last_status: null,
      enable_apply: false,
	  select_time_value: "min_5",
	  selected_time_option: { value: "min_5", label: i18n('show_alerts.presets.5_min'), currently_active: false },
      wrong_date: false,
      flat_begin_date: null,
      flat_end_date: null,
      time_preset_list: [
        { value: "min_5", label: i18n('show_alerts.presets.5_min'), currently_active: false },
        { value: "min_30", label: i18n('show_alerts.presets.30_min'), currently_active: true },
        { value: "hour", label: i18n('show_alerts.presets.hour'), currently_active: false },
        { value: "day", label: i18n('show_alerts.presets.day'), currently_active: false },
        { value: "week", label: i18n('show_alerts.presets.week'), currently_active: false },
        { value: "month", label: i18n('show_alerts.presets.month'), currently_active: false },
        { value: "year", label: i18n('show_alerts.presets.year'), currently_active: false },
        { value: "custom", label: i18n('show_alerts.presets.custom'), currently_active: false, disabled: true, },
      ]
    };
  },
}

</script>

<style scoped>
.date_time_input {
  width: 10.5rem;
  max-width: 10.5rem;
  min-width: 10.5rem;
}
</style>
