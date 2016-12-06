package main

import (
	"net/http"
	"log"
	"encoding/json"
	"io/ioutil"
)

type Json struct {
	//Message string `json:"message"`
	Data []Data `json:"data"`
}

type Data struct {
	Title string `json:"title"`
	Image_list []ImgList `json:"image_list"`
}

type ImgList struct {
	Url string `json:"url"`
}

func main() {
	host := "http://www.toutiao.com/search_content/?format=json&keyword=%E8%A1%97%E6%8B%8D&autoload=true&count=2"
	host += "&offset=0"

	resp, err := http.Get(host)
	if err != nil {
		log.Fatal(err)
	}

	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)

	if err != nil {
		log.Fatal(err)
	}

	var imgs Json
	json.Unmarshal([]byte(string(body)), &imgs)

	log.Println("imgs:",imgs)
}