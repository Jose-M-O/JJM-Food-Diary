package main

import (
	"encoding/json"
	"fmt"
	"log"
	"math"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const FoodsPath = "../data/foods.json"
const FoodLogPath = "../data/food_log.json"

type Food struct {
	FoodID   string  `json:"food_id"`
	Name     string  `json:"name"`
	KcalPerG float64 `json:"kcal_per_g"`
	FoodType string  `json:"food_type"`
}

type FoodEntry struct {
	EntryID  int     `json:"entry_id"`
	Date     string  `json:"date"`
	FoodID   string  `json:"food_id"`
	Name     string  `json:"name"`
	Grams    float64 `json:"grams"`
	KcalPerG float64 `json:"kcal_per_g"`
	FoodType string  `json:"food_type"`
	Calories float64 `json:"calories"`
}

type CreateFoodRequest struct {
	Name         string  `json:"name"`
	Calories     float64 `json:"calories"`
	ServingGrams float64 `json:"serving_grams"`
	FoodType     string  `json:"food_type"`
}

type AddEntryRequest struct {
	Date   string  `json:"date"`
	FoodID string  `json:"food_id"`
	Grams  float64 `json:"grams"`
}

type GraphNode struct {
	ID    string  `json:"id"`
	Label string  `json:"label"`
	Type  string  `json:"type"`
	Value float64 `json:"value"`
	Group string  `json:"group"`
}

type GraphLink struct {
	Source string `json:"source"`
	Target string `json:"target"`
}

type GraphResponse struct {
	DailyTotalCalories float64     `json:"daily_total_calories"`
	Nodes              []GraphNode `json:"nodes"`
	Links              []GraphLink `json:"links"`
}

type EntriesResponse struct {
	Date               string      `json:"date"`
	DailyTotalCalories float64     `json:"daily_total_calories"`
	Entries            []FoodEntry `json:"entries"`
}

func main() {
	if err := ensureDataFiles(); err != nil {
		log.Fatal(err)
	}

	http.HandleFunc("/api/foods", withCORS(handleFoods))
	http.HandleFunc("/api/foods/", withCORS(handleDeleteFood))
	http.HandleFunc("/api/entry", withCORS(handleAddEntry))
	http.HandleFunc("/api/entry/", withCORS(handleDeleteEntry))
	http.HandleFunc("/api/entries", withCORS(handleGetEntries))
	http.HandleFunc("/api/graph", withCORS(handleGraph))

	fs := http.FileServer(http.Dir("../frontend"))
	http.Handle("/", fs)

	log.Println("Server running at http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func withCORS(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Access-Control-Allow-Origin", "*")
		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, DELETE, OPTIONS")
		w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
		if r.Method == http.MethodOptions {
			w.WriteHeader(http.StatusOK)
			return
		}
		next(w, r)
	}
}

func ensureDataFiles() error {
	dataDir := "../data"
	if err := os.MkdirAll(dataDir, 0755); err != nil {
		return err
	}

	if _, err := os.Stat(FoodsPath); os.IsNotExist(err) {
		if err := writeJSON(FoodsPath, []Food{}); err != nil {
			return err
		}
	}

	if _, err := os.Stat(FoodLogPath); os.IsNotExist(err) {
		if err := writeJSON(FoodLogPath, []FoodEntry{}); err != nil {
			return err
		}
	}

	return nil
}

func readFoods() ([]Food, error) {
	var foods []Food
	err := readJSON(FoodsPath, &foods)
	return foods, err
}

func writeFoods(foods []Food) error {
	return writeJSON(FoodsPath, foods)
}

func readEntries() ([]FoodEntry, error) {
	var entries []FoodEntry
	err := readJSON(FoodLogPath, &entries)
	return entries, err
}

func writeEntries(entries []FoodEntry) error {
	return writeJSON(FoodLogPath, entries)
}

func readJSON(path string, target interface{}) error {
	file, err := os.ReadFile(path)
	if err != nil {
		return err
	}
	if len(file) == 0 {
		return nil
	}
	return json.Unmarshal(file, target)
}

func writeJSON(path string, data interface{}) error {
	bytes, err := json.MarshalIndent(data, "", "  ")
	if err != nil {
		return err
	}
	return os.WriteFile(path, bytes, 0644)
}

func normalize(s string) string {
	return strings.TrimSpace(strings.ToLower(s))
}

func titleCase(s string) string {
	if s == "" {
		return s
	}
	words := strings.Fields(strings.ToLower(s))
	for i, w := range words {
		if len(w) > 0 {
			words[i] = strings.ToUpper(w[:1]) + w[1:]
		}
	}
	return strings.Join(words, " ")
}

func round2(n float64) float64 {
	return math.Round(n*100) / 100
}

func nextFoodID(foods []Food) string {
	maxNum := 0
	for _, f := range foods {
		if strings.HasPrefix(f.FoodID, "food-") {
			numStr := strings.TrimPrefix(f.FoodID, "food-")
			num, err := strconv.Atoi(numStr)
			if err == nil && num > maxNum {
				maxNum = num
			}
		}
	}
	return fmt.Sprintf("food-%d", maxNum+1)
}

func nextEntryID(entries []FoodEntry) int {
	maxID := 0
	for _, e := range entries {
		if e.EntryID > maxID {
			maxID = e.EntryID
		}
	}
	return maxID + 1
}

func findFoodByID(foods []Food, id string) *Food {
	for _, f := range foods {
		if f.FoodID == id {
			copyFood := f
			return &copyFood
		}
	}
	return nil
}

func filterEntriesByDate(entries []FoodEntry, date string) []FoodEntry {
	result := []FoodEntry{}
	for _, e := range entries {
		if e.Date == date {
			result = append(result, e)
		}
	}
	return result
}

func buildGraph(entries []FoodEntry) GraphResponse {
	if len(entries) == 0 {
		return GraphResponse{
			DailyTotalCalories: 0,
			Nodes:              []GraphNode{},
			Links:              []GraphLink{},
		}
	}

	groupTotals := map[string]float64{}
	for _, e := range entries {
		groupTotals[e.FoodType] += e.Calories
	}

	total := 0.0
	for _, v := range groupTotals {
		total += v
	}
	total = round2(total)

	nodes := []GraphNode{}
	links := []GraphLink{}

	for group, calories := range groupTotals {
		percent := 0.0
		if total > 0 {
			percent = (calories / total) * 100
		}

		nodes = append(nodes, GraphNode{
			ID:    group,
			Label: fmt.Sprintf("%s\n%.2f cal\n%.1f%%", titleCase(group), round2(calories), percent),
			Type:  "group",
			Value: round2(calories),
			Group: group,
		})
	}

	for _, e := range entries {
		nodeID := fmt.Sprintf("entry-%d", e.EntryID)
		nodes = append(nodes, GraphNode{
			ID:    nodeID,
			Label: fmt.Sprintf("%s\n%.0fg\n%.2f cal", titleCase(e.Name), e.Grams, e.Calories),
			Type:  "food",
			Value: round2(e.Calories),
			Group: e.FoodType,
		})
		links = append(links, GraphLink{
			Source: e.FoodType,
			Target: nodeID,
		})
	}

	return GraphResponse{
		DailyTotalCalories: total,
		Nodes:              nodes,
		Links:              links,
	}
}

func handleFoods(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case http.MethodGet:
		foods, err := readFoods()
		if err != nil {
			http.Error(w, "Failed to read foods", http.StatusInternalServerError)
			return
		}

		query := normalize(r.URL.Query().Get("query"))
		if query != "" {
			filtered := []Food{}
			for _, f := range foods {
				if strings.Contains(normalize(f.Name), query) || strings.Contains(normalize(f.FoodType), query) {
					filtered = append(filtered, f)
				}
			}
			foods = filtered
		}

		writeJSONResponse(w, foods)

	case http.MethodPost:
		var req CreateFoodRequest
		if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
			http.Error(w, "Invalid JSON", http.StatusBadRequest)
			return
		}

		req.Name = strings.TrimSpace(req.Name)
		req.FoodType = normalize(req.FoodType)

		if req.Name == "" || req.Calories <= 0 || req.ServingGrams <= 0 || req.FoodType == "" {
			http.Error(w, "All fields are required and must be positive", http.StatusBadRequest)
			return
		}

		kcalPerG := round2(req.Calories / req.ServingGrams)

		foods, err := readFoods()
		if err != nil {
			http.Error(w, "Failed to read foods", http.StatusInternalServerError)
			return
		}

		for _, f := range foods {
			if normalize(f.Name) == normalize(req.Name) {
				http.Error(w, "Food already exists", http.StatusBadRequest)
				return
			}
		}

		newFood := Food{
			FoodID:   nextFoodID(foods),
			Name:     req.Name,
			KcalPerG: kcalPerG,
			FoodType: req.FoodType,
		}

		foods = append(foods, newFood)

		if err := writeFoods(foods); err != nil {
			http.Error(w, "Failed to save food", http.StatusInternalServerError)
			return
		}

		writeJSONResponse(w, newFood)

	default:
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
	}
}

func handleDeleteFood(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodDelete {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	foodID := strings.TrimPrefix(r.URL.Path, "/api/foods/")
	if foodID == "" {
		http.Error(w, "Food ID required", http.StatusBadRequest)
		return
	}

	foods, err := readFoods()
	if err != nil {
		http.Error(w, "Failed to read foods", http.StatusInternalServerError)
		return
	}

	filteredFoods := []Food{}
	found := false
	for _, f := range foods {
		if f.FoodID == foodID {
			found = true
			continue
		}
		filteredFoods = append(filteredFoods, f)
	}

	if !found {
		http.Error(w, "Food not found", http.StatusNotFound)
		return
	}

	if err := writeFoods(filteredFoods); err != nil {
		http.Error(w, "Failed to delete food", http.StatusInternalServerError)
		return
	}

	writeJSONResponse(w, map[string]bool{"success": true})
}

func handleAddEntry(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	var req AddEntryRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid JSON", http.StatusBadRequest)
		return
	}

	if strings.TrimSpace(req.Date) == "" {
		req.Date = time.Now().Format("2006-01-02")
	}

	if strings.TrimSpace(req.FoodID) == "" || req.Grams <= 0 {
		http.Error(w, "food_id and positive grams are required", http.StatusBadRequest)
		return
	}

	foods, err := readFoods()
	if err != nil {
		http.Error(w, "Failed to read foods", http.StatusInternalServerError)
		return
	}

	food := findFoodByID(foods, req.FoodID)
	if food == nil {
		http.Error(w, "Food not found", http.StatusNotFound)
		return
	}

	entries, err := readEntries()
	if err != nil {
		http.Error(w, "Failed to read entries", http.StatusInternalServerError)
		return
	}

	newEntry := FoodEntry{
		EntryID:  nextEntryID(entries),
		Date:     req.Date,
		FoodID:   food.FoodID,
		Name:     food.Name,
		Grams:    req.Grams,
		KcalPerG: food.KcalPerG,
		FoodType: food.FoodType,
		Calories: round2(req.Grams * food.KcalPerG),
	}

	entries = append(entries, newEntry)

	if err := writeEntries(entries); err != nil {
		http.Error(w, "Failed to save entry", http.StatusInternalServerError)
		return
	}

	writeJSONResponse(w, newEntry)
}

func handleDeleteEntry(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodDelete {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	entryIDStr := strings.TrimPrefix(r.URL.Path, "/api/entry/")
	entryID, err := strconv.Atoi(entryIDStr)
	if err != nil {
		http.Error(w, "Invalid entry ID", http.StatusBadRequest)
		return
	}

	entries, err := readEntries()
	if err != nil {
		http.Error(w, "Failed to read entries", http.StatusInternalServerError)
		return
	}

	filtered := []FoodEntry{}
	found := false
	for _, e := range entries {
		if e.EntryID == entryID {
			found = true
			continue
		}
		filtered = append(filtered, e)
	}

	if !found {
		http.Error(w, "Entry not found", http.StatusNotFound)
		return
	}

	if err := writeEntries(filtered); err != nil {
		http.Error(w, "Failed to delete entry", http.StatusInternalServerError)
		return
	}

	writeJSONResponse(w, map[string]bool{"success": true})
}

func handleGetEntries(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	date := strings.TrimSpace(r.URL.Query().Get("date"))
	if date == "" {
		date = time.Now().Format("2006-01-02")
	}

	entries, err := readEntries()
	if err != nil {
		http.Error(w, "Failed to read entries", http.StatusInternalServerError)
		return
	}

	dayEntries := filterEntriesByDate(entries, date)
	total := 0.0
	for _, e := range dayEntries {
		total += e.Calories
	}

	resp := EntriesResponse{
		Date:               date,
		DailyTotalCalories: round2(total),
		Entries:            dayEntries,
	}

	writeJSONResponse(w, resp)
}

func handleGraph(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	date := strings.TrimSpace(r.URL.Query().Get("date"))
	if date == "" {
		date = time.Now().Format("2006-01-02")
	}

	entries, err := readEntries()
	if err != nil {
		http.Error(w, "Failed to read entries", http.StatusInternalServerError)
		return
	}

	dayEntries := filterEntriesByDate(entries, date)
	graph := buildGraph(dayEntries)
	writeJSONResponse(w, graph)
}

func writeJSONResponse(w http.ResponseWriter, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(data)
}
