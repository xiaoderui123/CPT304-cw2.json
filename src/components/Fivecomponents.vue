<template>
    <div>
        <h2>Select a country:</h2>
        <select v-model="selectedCountry" @change="fetchData" >
            <option v-for="country in countries" :value="country.code">
                {{ country.name }} - {{ country.code }} ({{ country.latitude }}, {{ country.longitude }})
            </option>
        </select>

        <h2>Select a holiday:</h2>
        <select v-model="selectedHoliday">
            <option v-for="holiday in holidays" :value="holiday.date">
                {{ holiday.name }} - {{ holiday.date }}
            </option>
        </select>

        <h2>Select a place:</h2>
        <select v-model="selectedCity">
            <option v-for="city in cities" :value="city">
                {{ city.name }}
            </option>
        </select>



        <h2>Weather for {{ selectedHoliday }} in {{ selectedCity.name }}:</h2>
        <p v-if="weather">
            Weather situation: {{ weather}} - Temperature: {{ tem }} °C
        </p>


        <h2>Available accommodation for {{ selectedHoliday }} in {{ selectedCity.name }}:</h2>
        <select v-model="accommodation.length">
            <option v-for="place in accommodation" :key="place.id">
                {{ place.name }} - {{ place.price.lead.formatted }} per night
            </option>
        </select>
    </div>
</template>

<script>
import axios from 'axios';

export default {
    data() {
        return {
            countries: [],
            selectedCountry: '',
            cities: [],
            selectedCity: '',
            holidays: [],
            selectedHoliday: '',
            weather: '',
            accommodation: [],
            latitude: '',
            longitude: '',
            tem:'',
            cityid:'',
        };
    },
    mounted() {
        this.fetchCountries();
    },
    methods: {
        async fetchData() {
            this.fetchHolidays();
            this.fetchCities();
        },
        async fetchCountries() {
            try {
                const response = await axios.get('https://restcountries.com/v3.1/all');
                this.countries = response.data.map(country => {
                    return {
                        name: country.name.common,
                        code: country.cca2,
                        latitude: country.latlng[0], // 获取国家的纬度信息
                        longitude: country.latlng[1], // 获取国家的经度信息
                    };
                }).sort((a, b) => {
                    // 使用 localeCompare 方法按照国家名称的首字母进行排序
                    return a.name.localeCompare(b.name);
                });
            } catch (error) {
                console.error('Failed to fetch countries:', error);
            }
        },

        async fetchHolidays() {
            const apiUrl = `https://date.nager.at/api/v3/PublicHolidays/2023/${this.selectedCountry}`;
            axios.get(apiUrl)
                .then(response => {
                    console.log(response)
                    this.holidays = response.data;
                })
                .catch(error => {
                    console.error(error);
                })
                .finally(() => {
                    this.loading = false;
                });
        },
        async fetchCities() {
            try {
                const response = await axios.get(`http://api.geonames.org/searchJSON?username=xdrxdr&country=${this.selectedCountry}`);
                console.log(response);
                this.cities = response.data.geonames
                    .map(city => {
                        return {
                            name: city.adminName1,
                        };
                    })
                    .filter((city, index, self) => {
                        // 使用 indexOf 判断元素在数组中的第一个位置是否等于当前位置
                        return self.findIndex(item => item.name === city.name) === index;
                    })
                    .filter(city => Boolean(city.name)); // 使用 filter 方法过滤掉空值
            } catch (error) {
                console.error(error);
            }
        },

        async fetchWeather() {
            const response = await axios.get(`http://api.weatherapi.com/v1/current.json?key=c6f1d2031d23456fb3255912232004&q=${this.selectedCity.name}&aqi=no`);
            console.log(response);
            this.weather = response.data.current.condition.text
            this.tem=response.data.current.temp_c
        },
        async fetchAccommodation() {
            const option1 = {
                method: 'GET',
                url: 'https://hotels-com-provider.p.rapidapi.com/v2/regions',
                params: {
                    locale: 'en_GB',
                    query: this.selectedCity.name,
                    domain: 'AE',
                },
                headers: {
                    'X-RapidAPI-Key': '8492f53e4emshafff7a254de76e9p1ca211jsn08d43f7cd931',
                    'X-RapidAPI-Host': 'hotels-com-provider.p.rapidapi.com'
                }
            };
            const option2 = {
                method: 'GET',
                url: 'https://hotels-com-provider.p.rapidapi.com/v2/hotels/search',
                params: {
                    domain: 'AE',
                    sort_order: 'REVIEW',
                    locale: 'en_GB',
                    checkout_date: '2023-12-31',
                    region_id: '',
                    adults_number: '1',
                    checkin_date: '',
                    available_filter: 'SHOW_AVAILABLE_ONLY',
                    meal_plan: 'FREE_BREAKFAST',
                    guest_rating_min: '8',
                    price_min: '10',
                    page_number: '1',
                    children_ages: '4,0,15',
                    amenities: 'WIFI,PARKING',
                    price_max: '500',
                    lodging_type: 'HOTEL,HOSTEL,APART_HOTEL',
                    payment_type: 'PAY_LATER,FREE_CANCELLATION',
                    star_rating_ids: '3,4,5'
                },
                headers: {
                    'X-RapidAPI-Key': '8492f53e4emshafff7a254de76e9p1ca211jsn08d43f7cd931',
                    'X-RapidAPI-Host': 'hotels-com-provider.p.rapidapi.com'
                }
            };
            try {
                const response1 = await axios.request(option1);
                console.log(response1);
                this.cityid=response1.data.data[0].gaiaId
            } catch (error) {
                console.error(error);
            }
            try {
                option2.params.checkin_date=this.selectedHoliday
                option2.params.region_id=this.cityid
                const response2 = await axios.request(option2);
                console.log(response2);
                this.accommodation=response2.data.properties
            } catch (error) {
                console.error(error);
            }


        },
    },
    watch: {
        selectedCity() {
            this.fetchWeather();
            this.fetchAccommodation();
        },

    },
};
</script>
