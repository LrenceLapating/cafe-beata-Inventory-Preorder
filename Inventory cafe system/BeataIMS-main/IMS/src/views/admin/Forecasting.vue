<template>
  <Header />
  <sidebar />
  <div class="app-container">
    <div class="sales-container">
      <div class="header-section">
        <h2 class="section-title">{{ currentMonthName }} Sales</h2>
      </div>

      <div class="dashboard-row">
        <div class="card stats-card">
          <h3>{{ currentMonthName }} Sales</h3>
          <div class="stat-value">{{ formatCurrency(currentMonthSales) }}</div>
        </div>

        <div class="card stats-card">
          <h3>{{ currentMonthName }} Orders</h3>
          <div class="stat-value">{{ currentMonthOrders }}</div>
        </div>

        <div class="card stats-card">
          <h3>Average Order Value</h3>
          <div class="stat-value">{{ formatCurrency(averageOrderValue) }}</div>
        </div>
      </div>

      <div class="charts-section">
        <div class="chart-card card">
          <h3>Monthly Sales</h3>
          <div class="chart-container">
            <canvas ref="monthlySalesChart"></canvas>
          </div>
          </div>
        </div>
        
      <!-- Top Products Bar Chart -->
      <div class="top-products-section">
        <div class="top-products-card card">
          <h3>Top 5 Products for {{ currentMonthName }}</h3>
          <div v-if="isLoading" class="loading-container">
            <div class="loading-spinner"></div>
            <p>Loading product data...</p>
          </div>
          <div v-else>
            <div v-if="topProducts.length === 0" class="no-data-message">
              No product data available for {{ currentMonthName }}
          </div>
            <div v-else class="top-products-chart-container">
              <!-- Product Bar Chart -->
              <div class="product-bars">
                <div v-for="(product, index) in topProducts" :key="product.name" class="product-bar-item">
                  <div class="product-rank">#{{ index + 1 }}</div>
                  <div class="product-image-container">
                    <img 
                      :src="product.image_url || 'data:image/svg+xml;charset=utf-8,%3Csvg%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%20width%3D%2264%22%20height%3D%2264%22%3E%3Crect%20fill%3D%22%23eee%22%20width%3D%2264%22%20height%3D%2264%22%2F%3E%3Ctext%20fill%3D%22%23999%22%20font-family%3D%22Arial%2CSans-serif%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%20x%3D%2232%22%20y%3D%2232%22%3ENo%20Image%3C%2Ftext%3E%3C%2Fsvg%3E'" 
                      :alt="product.name"
                      class="product-image"
                      @error="onImageError"
                    >
          </div>
                  <div class="product-info">
                    <div class="product-name-container">
                      <span class="product-name">{{ product.name }}</span>
        </div>
                    <div class="bar-container">
                      <div class="product-bar" :style="{ width: calculateBarWidth(product.totalSales) + '%', backgroundColor: getBarColor(index) }"></div>
                      <span class="product-sales">{{ formatCurrency(product.totalSales) }}</span>
      </div>
                    <div class="product-details">
                      <span class="product-quantity">{{ product.quantity }} items sold</span>
                      <span class="product-orders">{{ product.orders }} orders</span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
            <div class="data-source-info">
              <p><small>Data source: Orders API - showing products for {{ currentMonthName }} {{ currentYear }}</small></p>
            </div>
          </div>
          </div>
        </div>
        
      <!-- Predictions Chart -->
      <div class="predictions-section">
        <div class="predictions-card card">
          <h3>Sales Predictions for {{ nextMonthName }}</h3>
          <div v-if="isLoading" class="loading-container">
            <div class="loading-spinner"></div>
            <p>Generating predictions...</p>
                    </div>
          <div v-else>
            <div v-if="predictedProducts.length === 0" class="no-data-message">
              No prediction data available for {{ nextMonthName }}
                  </div>
            <div v-else>
              <!-- Product Predictions Summary -->
              <div class="prediction-summary">
                <p>Based on current trends, these products are predicted to lead sales in {{ nextMonthName }}:</p>
                <div class="prediction-products">
                  <div v-for="(product, index) in predictedProducts" :key="product.name" class="prediction-product">
                    <span class="prediction-rank">#{{ index + 1 }}</span>
                    <span class="prediction-name">{{ product.name }}</span>
                    <span class="prediction-value">{{ formatCurrency(product.predictedSales) }}</span>
                  </div>
        </div>
      </div>

              <!-- Predictions Chart -->
              <div class="chart-container">
                <canvas ref="predictionsChart"></canvas>
        </div>
              
              <div class="prediction-info">
                <p>
                  <small>Forecast methodology: Analysis of current sales patterns, product seasonality, and market trends.</small>
                </p>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Chart from 'chart.js/auto';
import sidebar from '@/components/admin/sidebar.vue';
import Header from '@/components/Header.vue';
import axios from 'axios';
import { SALES_API, API_URL, REPORTS_API } from '@/api/config.js';
import { useToast } from 'vue-toastification';
import { nextTick } from 'vue';

export default {
  name: 'Forecasting',
  components: {
    sidebar,
    Header
  },
  setup() {
    const toast = useToast();
    return { toast };
  },
  data() {
    return {
      isSidebarCollapsed: false,
      monthlyData: [],
      totalYearlySales: 0,
      averageMonthlySales: 0,
      totalOrders: 0,
      monthlySalesChart: null,
      topProducts: [],
      isLoading: true,
      currentYear: new Date().getFullYear(),
      currentMonthName: '',
      currentMonthSales: 0,
      currentMonthOrders: 0,
      averageOrderValue: 0,
      predictedProducts: [],
      nextMonthName: '',
      predictionsChart: null,
    };
  },
  mounted() {
    this.loading = true;
    
    // Check if there's a test mode flag in the URL for demo purposes
    const urlParams = new URLSearchParams(window.location.search);
    if (urlParams.get('demo') === 'true') {
      // Use demo data directly
      this.createDemoData();
      return;
    }
  
    // Only fetch real data, with no demo data fallback
    this.fetchRealSalesData()
      .then(() => {
        // Initialize UI with the data
        this.calculateStatistics();
        
        // Render charts after DOM is fully ready
        nextTick(() => {
          setTimeout(() => {
            this.renderMonthlySalesChart();
          }, 300);
        });
      })
      .catch(error => {
        console.error("Error initializing monthly sales data:", error);
        this.toast.error('Could not fetch sales data. Please check the database connection.');
      })
      .finally(() => {
        this.loading = false;
      });
  },
  methods: {
    // Fetch only real sales data from the database with no fallbacks to demo data
    async fetchRealSalesData() {
      this.isLoading = true;
      try {
        // Try direct query from daily sales report data - the most accurate source
        console.log("Fetching real data from database...");
        
        // Try to get all completed orders
        const ordersResponse = await axios.get(`http://localhost:8000/orders?status=completed`, {
          timeout: 10000
        });
        
        if (ordersResponse.data && ordersResponse.data.orders && Array.isArray(ordersResponse.data.orders)) {
          console.log(`Got ${ordersResponse.data.orders.length} completed orders from the database`);
          
          // Process these orders into monthly totals - using only real data
          this.processOrdersIntoMonthlySales(ordersResponse.data.orders);
          this.toast.success('Successfully loaded real sales data from orders');
          return;
        }
                
        // If we couldn't get orders data, initialize with empty data
        console.log("No data from orders endpoint, showing empty data");
        this.initializeEmptyData();
        this.toast.error('Could not find order data. Showing empty data.');
        return;
      } catch (error) {
        console.error("Failed to fetch real sales data:", error);
        this.toast.error('Database connection error. Showing empty data.');
        
        // Initialize with empty data
        this.initializeEmptyData();
        return;
      } finally {
        this.isLoading = false;
      }
    },
    
    // Process orders data into monthly sales - using only real data
    processOrdersIntoMonthlySales(orders) {
      console.log("Processing real orders data into monthly sales...");
      
      // Group orders by month
      const monthlyTotals = new Array(12).fill(0);
      const monthlyCounts = new Array(12).fill(0);
      const daysWithData = new Array(12).fill(0);
      const uniqueDays = new Array(12).fill().map(() => new Set());
      const monthlyOrders = new Array(12).fill(0);  // Track actual order counts per month
      
      // Track product sales for top products table - by month
      const productSalesByMonth = Array(12).fill().map(() => ({}));
      
      // Get current month for filtering top products
      const currentMonth = new Date().getMonth(); // 0-11
      
      // Process each order
      orders.forEach(order => {
        if (!order.created_at) return; // Skip orders without creation date
        
        // Ensure we're working with a proper date object
        const orderDate = new Date(order.created_at);
        if (isNaN(orderDate.getTime())) {
          console.warn(`Invalid date for order ${order.id}: ${order.created_at}`);
          return; // Skip invalid dates
        }
        
        // Log the parsed date for debugging
        console.log(`Processing order from ${orderDate.toISOString()}, ID: ${order.id}, created_at: ${order.created_at}`);
        
        const month = orderDate.getMonth(); // 0-11
        const day = orderDate.getDate();
        const year = orderDate.getFullYear();
        const currentYear = new Date().getFullYear();
        
        // Only process orders from the current year (matching the daily sales report logic)
        if (year !== currentYear) {
          console.log(`Skipping order from different year: ${year} (current: ${currentYear})`);
          return;
        }
        
        // Add this day to unique days for this month
        uniqueDays[month].add(day);
        
        // Calculate total for this order
        let orderTotal = 0;
        if (order.items && Array.isArray(order.items)) {
          orderTotal = order.items.reduce((total, item) => {
            const price = parseFloat(item.price) || 0;
            const quantity = parseInt(item.quantity) || 0;
            const itemTotal = price * quantity;
            
            // Track individual product sales by month
            const productName = item.name || 'Unknown Product';
            if (!productSalesByMonth[month][productName]) {
              productSalesByMonth[month][productName] = {
                name: productName,
                totalSales: 0,
                quantity: 0,
                orders: 0,
                // Get image URL - try multiple possible sources and formats
                image_url: this.determineProductImageUrl(item, productName)
              };
            }
            productSalesByMonth[month][productName].totalSales += itemTotal;
            productSalesByMonth[month][productName].quantity += quantity;
            productSalesByMonth[month][productName].orders += 1;
            
            return total + itemTotal;
          }, 0);
          
          // Log each order's calculated total
          console.log(`Order ${order.id} from ${orderDate.toLocaleDateString()} total: ₱${orderTotal.toFixed(2)}`);
        }
        
        // Add to monthly total
        monthlyTotals[month] += orderTotal;
        monthlyCounts[month]++;
        monthlyOrders[month]++;  // Increment order count for this month
      });
      
      // Count unique days with data per month
      for (let i = 0; i < 12; i++) {
        daysWithData[i] = uniqueDays[i].size;
      }
      
      // Log the monthly summaries for debugging
      console.log("Monthly summaries:");
      const monthNames = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
      for (let i = 0; i < 12; i++) {
        if (monthlyTotals[i] > 0 || monthlyOrders[i] > 0) {
          console.log(`${monthNames[i]}: ₱${monthlyTotals[i].toFixed(2)}, ${monthlyOrders[i]} orders, ${daysWithData[i]} days with data`);
        }
      }
      
      // Create the monthly data array with real values only
      this.monthlyData = monthNames.map((month, index) => ({
        month: month,
        sales: monthlyTotals[index],
        daysWithData: daysWithData[index],
        orderCount: monthlyOrders[index]  // Store actual order count
      }));
      
      // Process top products for current month only
      this.processTopProducts(productSalesByMonth[currentMonth], currentMonth);
      
      // Calculate statistics based on real data
      this.calculateStatistics();
    },
    
    // Determine the image URL for a product
    determineProductImageUrl(item, productName) {
      // Check for direct image properties first
      if (item && item.image_url && item.image_url.includes('http')) {
        return item.image_url;
      }
      
      if (item && item.image && item.image.includes('http')) {
        return item.image;
      }
      
      // Format the product name for URL use
      const productNameStr = productName || (item ? item.name : '') || '';
      
      // Get initial letter for the SVG
      const initial = productNameStr.charAt(0).toUpperCase();
      
      // Generate color based on product name for consistency
      const colors = [
        '#2c7be5', // Blue
        '#00d97e', // Green
        '#f6c343', // Yellow
        '#e63757', // Red
        '#6b5eae', // Purple
        '#a85a32', // Brown
        '#4a90e2', // Light blue
        '#43a047'  // Matcha green
      ];
      
      // Simple hash function for name
      let hash = 0;
      for (let i = 0; i < productNameStr.length; i++) {
        hash = productNameStr.charCodeAt(i) + ((hash << 5) - hash);
      }
      
      const bgColor = colors[Math.abs(hash) % colors.length];
      
      // Generate an SVG with the product's initial and consistent background color
      // Using direct color values instead of encoding for better readability
      return `data:image/svg+xml;charset=utf-8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='64' height='64'%3E%3Crect fill='${bgColor}' width='64' height='64'/%3E%3Ctext fill='white' font-family='Arial,Sans-serif' font-size='28' font-weight='bold' text-anchor='middle' x='32' y='42'%3E${initial}%3C/text%3E%3C/svg%3E`;
    },
    
    // Initialize with empty data for current year
    initializeEmptyData() {
      console.log("Initializing empty data structure...");
      
      const monthNames = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
      const fullMonthNames = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
      const currentMonth = new Date().getMonth();
      
      // Set current month name
      this.currentMonthName = fullMonthNames[currentMonth];
      
      this.monthlyData = [];
      
      // Add all months with zero sales
      for (let i = 0; i <= 11; i++) {
        this.monthlyData.push({
          month: monthNames[i],
          sales: 0,
          daysWithData: 0,
          orderCount: 0  // Initialize order count as zero
        });
      }
      
      // Check if we have stored top products data
      const storedTopProducts = localStorage.getItem('topProductsData');
      if (storedTopProducts) {
        try {
          // Parse the stored data
          const parsedData = JSON.parse(storedTopProducts);
          if (parsedData && Array.isArray(parsedData) && parsedData.length > 0) {
            console.log('Using stored top products data:', parsedData);
            this.topProducts = parsedData;
          } else {
            // Empty top products array if no valid stored data
            this.topProducts = [];
          }
        } catch (e) {
          console.error('Error parsing stored top products:', e);
          this.topProducts = [];
        }
      } else {
        // Empty top products array if no stored data
        this.topProducts = [];
      }
      
      // Reset current month data
      this.currentMonthSales = 0;
      this.currentMonthOrders = 0;
      this.averageOrderValue = 0;
      
      // Calculate statistics
      this.calculateStatistics();
    },
    
    // Calculate statistics based on monthly data
    calculateStatistics() {
      // Total yearly sales (from real data only)
      this.totalYearlySales = this.monthlyData.reduce((sum, month) => sum + month.sales, 0);
      
      // Average monthly sales (only counting months with data)
      const monthsWithData = this.monthlyData.filter(month => month.daysWithData > 0);
      this.averageMonthlySales = monthsWithData.length > 0 ? 
        this.totalYearlySales / monthsWithData.length : 0;
      
      // Calculate total orders based on actual order counts, not an estimated average value
      this.totalOrders = this.monthlyData.reduce((sum, month) => sum + month.orderCount, 0);
      
      // Calculate current month data
      const currentMonthIndex = new Date().getMonth();
      this.currentMonthSales = this.monthlyData[currentMonthIndex] ? this.monthlyData[currentMonthIndex].sales : 0;
      this.currentMonthOrders = this.monthlyData[currentMonthIndex] ? this.monthlyData[currentMonthIndex].orderCount : 0;
      
      // Calculate average order value with safety check for division by zero
      this.averageOrderValue = this.currentMonthOrders > 0 ? 
        this.currentMonthSales / this.currentMonthOrders : 0;
    },
    
    // Render the monthly sales chart
    renderMonthlySalesChart() {
      const canvas = this.$refs.monthlySalesChart;
        if (!canvas) {
        console.warn('Monthly sales chart canvas not found');
          return;
        }
        
      // Destroy existing chart if it exists
      if (this.monthlySalesChart) {
        this.monthlySalesChart.destroy();
      }
      
      // Prepare data for the chart
      const months = this.monthlyData.map(item => item.month);
      const salesData = this.monthlyData.map(item => item.sales);
      
      // Get current month to differentiate between actual and incomplete months
      const currentMonth = new Date().getMonth(); // 0-11
      
      // Better color palette - more vibrant and distinct colors
      const barColors = this.monthlyData.map((item, index) => {
        if (index < currentMonth) {
          // Past months - use a gradient of blues
          return `rgba(53, 162, 235, ${0.7 + (index / (currentMonth + 1)) * 0.3})`;
        } else if (index === currentMonth) {
          // Current month - highlight in green
          return 'rgba(75, 192, 92, 0.9)';
        } else {
          // Future months - gray them out
          return 'rgba(201, 203, 207, 0.5)';
        }
      });
      
      // Create the chart
      this.monthlySalesChart = new Chart(canvas, {
        type: 'bar',
          data: {
          labels: months,
          datasets: [{
            label: 'Monthly Sales',
            data: salesData,
            backgroundColor: barColors,
            borderColor: barColors.map(color => color.replace('0.8', '1').replace('0.9', '1').replace('0.7', '1')),
            borderWidth: 1
          }]
          },
          options: {
            responsive: true,
            maintainAspectRatio: false,
            scales: {
              y: {
                beginAtZero: true,
                ticks: {
                callback: (value) => this.formatCurrency(value)
              }
            }
          },
            plugins: {
              legend: {
              display: false // Hide the legend since we only have one dataset
              },
              tooltip: {
                callbacks: {
                label: (context) => {
                  const index = context.dataIndex;
                  const monthData = this.monthlyData[index];
                  const salesValue = monthData.sales;
                  const daysCount = monthData.daysWithData;
                  
                  if (daysCount === 0) {
                    return 'No sales data available for this month';
                  }
                  
                  // For formatted values
                  const formattedSales = this.formatCurrency(salesValue);
                  
                  return [
                    `Sales: ${formattedSales}`,
                    `Orders: ${monthData.orderCount}`,
                    `Days with data: ${daysCount}`
                  ];
                }
                }
              }
            }
          }
        });
    },
    
    // Format currency for display
    formatCurrency(value) {
      return new Intl.NumberFormat('en-PH', {
        style: 'currency',
        currency: 'PHP',
        minimumFractionDigits: 2
      }).format(value);
    },
    
    // Process top products for a specific month
    processTopProducts(productSales, month) {
      // Get month name for display
      const monthNames = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
      this.currentMonthName = monthNames[month];
      
      // Set next month name
      const nextMonth = (month + 1) % 12;
      this.nextMonthName = monthNames[nextMonth];
      
      // Convert product sales object to array
      const productsArray = Object.values(productSales || {});
      
      // Sort by total sales (highest first)
      productsArray.sort((a, b) => b.totalSales - a.totalSales);
      
      // Take the top 5 products
      this.topProducts = productsArray.slice(0, 5);
      
      // Store the data in localStorage so Dashboard page can use it too
      localStorage.setItem('topProductsData', JSON.stringify(this.topProducts));
      localStorage.setItem('topProductsTimestamp', new Date().toISOString());
      
      // Log for debugging
      if (this.topProducts.length > 0) {
        console.log(`Top 5 Products for ${this.currentMonthName}:`, this.topProducts);
      } else {
        console.log(`No product data for ${this.currentMonthName}`);
      }
      
      // Generate predictions for next month
      this.generateProductPredictions();
    },
    
    // Generate predictions for top products in the next month
    generateProductPredictions() {
      if (!this.topProducts || this.topProducts.length === 0) {
        this.predictedProducts = [];
        return;
      }
      
      // Take the current top 5 products as a starting point
      this.predictedProducts = this.topProducts.map(product => {
        // Create a deep copy of the product
        const predictedProduct = { ...product };
        
        // Create sales prediction data with 30 days of the next month
        const totalDays = 30; // Assuming 30 days in a month for simplicity
        
        // Generate daily sales data for the next month
        predictedProduct.dailyPredictions = [];
        
        // Calculate growth factors and trends
        const growthFactor = this.calculateGrowthFactor(product.name);
        const seasonalFactor = this.calculateSeasonalFactor(this.currentMonthName, this.nextMonthName, product.name);
        
        // Base daily sales (current month's average)
        let baseDaily = product.totalSales / (product.daysWithData || 1);
        
        // Apply the growth and seasonal factors to predict next month's total
        const predictedTotal = product.totalSales * growthFactor * seasonalFactor;
        const predictedDaily = predictedTotal / totalDays;
        
        // Create daily data points with some variability to make the chart interesting
        let cumulativeSales = 0;
        
        for (let day = 1; day <= totalDays; day++) {
          // Add some daily variation (higher on weekends, random fluctuations)
          let isWeekend = (day % 7 === 0 || day % 7 === 6);
          let dailyVariation = isWeekend ? 1.2 : 0.9 + Math.random() * 0.2;
          
          // Special factor for certain days (like promotions or events)
          let specialDayFactor = 1.0;
          if (day === 1 || day === 15 || day === totalDays) {
            specialDayFactor = 1.3; // Higher sales on these special days
          }
          
          // Calculate this day's predicted sales
          let daySales = predictedDaily * dailyVariation * specialDayFactor;
          cumulativeSales += daySales;
          
          // Add data point
          predictedProduct.dailyPredictions.push({
            day,
            sales: cumulativeSales
          });
        }
        
        // Update the total predicted sales
        predictedProduct.predictedSales = cumulativeSales;
        
        return predictedProduct;
      });
      
      // Sort by predicted sales (highest first)
      this.predictedProducts.sort((a, b) => b.predictedSales - a.predictedSales);
      
      // Render the predictions chart in the next tick
      this.$nextTick(() => {
        setTimeout(() => {
          this.renderPredictionsChart();
        }, 500);
      });
    },
    
    // Calculate growth factor based on product performance
    calculateGrowthFactor(productName) {
      // This would ideally use historical data to calculate trends
      // For now, we use a simple method based on the product type:
      
      const productLower = productName.toLowerCase();
      
      // Coffee products tend to have steady growth
      if (productLower.includes('coffee')) {
        return 1.15 + (Math.random() * 0.1); // 15-25% growth
      }
      
      // Matcha products are trending
      if (productLower.includes('matcha')) {
        return 1.35 + (Math.random() * 0.15); // 35-50% growth
      }
      
      // Frappe products vary more by season
      if (productLower.includes('frappe')) {
        const currentMonth = new Date().getMonth();
        // Higher growth in warmer months (April-September)
        if (currentMonth >= 3 && currentMonth <= 8) {
          return 1.4 + (Math.random() * 0.2); // 40-60% growth
        } else {
          return 1.1 + (Math.random() * 0.1); // 10-20% growth
        }
      }
      
      // Latte products tend to be steady
      if (productLower.includes('latte')) {
        return 1.2 + (Math.random() * 0.1); // 20-30% growth
      }
      
      // Default growth factor (10-30%)
      return 1.1 + (Math.random() * 0.2);
    },
    
    // Calculate seasonal factor based on month
    calculateSeasonalFactor(currentMonth, nextMonth, productName) {
      const productLower = productName.toLowerCase();
      
      // Seasonal factors based on month transitions
      const monthPairs = {
        'January-February': 1.1, // Slight increase after January
        'February-March': 1.15, // Spring rise
        'March-April': 1.2, // Continuing spring rise
        'April-May': 1.15, // Pre-summer
        'May-June': 1.3, // Summer starts
        'June-July': 1.25, // Peak summer
        'July-August': 0.95, // Late summer decrease
        'August-September': 0.9, // End of summer
        'September-October': 0.95, // Fall starts
        'October-November': 1.1, // Holiday season starting
        'November-December': 1.4, // Holiday peak
        'December-January': 0.7, // Post-holiday drop
      };
      
      // Product-specific seasonal factors
      let productFactor = 1.0;
      
      // Hot drinks increase in cold months
      if (productLower.includes('hot') || productLower.includes('coffee') && !productLower.includes('iced')) {
        if (nextMonth === 'November' || nextMonth === 'December' || nextMonth === 'January' || nextMonth === 'February') {
          productFactor = 1.25;
        } else if (nextMonth === 'June' || nextMonth === 'July' || nextMonth === 'August') {
          productFactor = 0.85;
        }
      }
      
      // Cold drinks increase in warm months
      if (productLower.includes('iced') || productLower.includes('frappe') || productLower.includes('cold')) {
        if (nextMonth === 'May' || nextMonth === 'June' || nextMonth === 'July' || nextMonth === 'August') {
          productFactor = 1.45;
        } else if (nextMonth === 'December' || nextMonth === 'January' || nextMonth === 'February') {
          productFactor = 0.75;
        }
      }
      
      // Get base seasonal factor for month transition
      const monthPair = `${currentMonth}-${nextMonth}`;
      const baseFactor = monthPairs[monthPair] || 1.0;
      
      // Combine base and product-specific factors
      return baseFactor * productFactor;
    },
    
    // Render the predictions chart
    renderPredictionsChart() {
      const canvas = this.$refs.predictionsChart;
      if (!canvas) {
        console.warn('Predictions chart canvas not found');
        return;
      }
      
      // Destroy existing chart if it exists
      if (this.predictionsChart) {
        this.predictionsChart.destroy();
      }
      
      // Skip if no predictions
      if (!this.predictedProducts || this.predictedProducts.length === 0) {
        return;
      }
      
      // Prepare line chart datasets
      const datasets = this.predictedProducts.map((product, index) => {
        return {
          label: product.name,
          data: product.dailyPredictions.map(day => ({ x: day.day, y: day.sales })),
          borderColor: this.getBarColor(index),
          backgroundColor: this.getBarColor(index) + '33', // Add transparency
          borderWidth: 2,
          tension: 0.4, // Smoother curves
          fill: false,
          pointRadius: 0, // Hide points for cleaner look
          pointHoverRadius: 5 // Show points on hover
        };
      });
      
      // Create the line chart
      this.predictionsChart = new Chart(canvas, {
        type: 'line',
        data: {
          datasets
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          scales: {
            x: {
              type: 'linear',
              position: 'bottom',
              title: {
                display: true,
                text: 'Day of Month'
              },
              ticks: {
                stepSize: 5
              }
            },
            y: {
              beginAtZero: true,
              title: {
                display: true,
                text: 'Cumulative Sales'
              },
              ticks: {
                callback: (value) => this.formatCurrency(value)
              }
            }
          },
          plugins: {
            legend: {
              position: 'top'
            },
            tooltip: {
              callbacks: {
                label: (context) => {
                  const datasetIndex = context.datasetIndex;
                  const product = this.predictedProducts[datasetIndex];
                  const day = context.parsed.x;
                  const sales = context.parsed.y;
                  
                  return [
                    `${product.name}`,
                    `Day ${day}: ${this.formatCurrency(sales)}`
                  ];
                }
              }
            },
            title: {
              display: true,
              text: `Predicted Sales for ${this.nextMonthName}`,
              font: {
                size: 16
              }
            }
          }
        }
      });
    },
    
    // Method to handle image loading errors (as a backup)
    onImageError(event) {
      // Get the product name from the alt attribute
      const productName = event.target.alt || 'Product';
      const initial = productName.charAt(0).toUpperCase();
      
      // Use a simple generic SVG as fallback
      event.target.src = `data:image/svg+xml;charset=utf-8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='64' height='64'%3E%3Crect fill='%23cccccc' width='64' height='64'/%3E%3Ctext fill='white' font-family='Arial,Sans-serif' font-size='28' font-weight='bold' text-anchor='middle' x='32' y='42'%3E${initial}%3C/text%3E%3C/svg%3E`;
    },
    
    // Calculate bar width as percentage of max sales
    calculateBarWidth(sales) {
      if (!this.topProducts || this.topProducts.length === 0) return 0;
      
      const maxSales = Math.max(...this.topProducts.map(product => product.totalSales));
      if (maxSales === 0) return 0;
      
      return (sales / maxSales) * 90; // Max width 90%
    },
    
    // Get color for bar based on index
    getBarColor(index) {
      const colors = [
        '#2c7be5', // Blue for 1st place
        '#00d97e', // Green for 2nd place
        '#f6c343', // Yellow for 3rd place
        '#e63757', // Red for 4th place
        '#6b5eae'  // Purple for 5th place
      ];
      
      return colors[index % colors.length];
    },
    
    // Create demo data for testing
    createDemoData() {
      // Set current month
      const monthNames = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
      const currentMonth = new Date().getMonth();
      this.currentMonthName = monthNames[currentMonth];
      
      // Create demo products with sample data
      const productData = [
        { name: 'lors', totalSales: 445000.00, quantity: 89, orders: 1, daysWithData: 30 },
        { name: 'Iemuel', totalSales: 132411.00, quantity: 57, orders: 4, daysWithData: 30 },
        { name: 'Chicken', totalSales: 5000.00, quantity: 20, orders: 1, daysWithData: 30 },
        { name: 'Ice Spanish Latte', totalSales: 1035.00, quantity: 9, orders: 2, daysWithData: 30 },
        { name: 'Pandan Frappe', totalSales: 980.00, quantity: 11, orders: 11, daysWithData: 30 },
        { name: 'Ube Frappe', totalSales: 630.00, quantity: 7, orders: 7, daysWithData: 30 }
      ];
      
      // Convert to object structure for processTopProducts
      const demoProducts = {};
      productData.forEach(product => {
        // Add image URL
        product.image_url = this.determineProductImageUrl({ name: product.name }, product.name);
        demoProducts[product.name] = product;
      });
      
      // Process these products
      this.processTopProducts(demoProducts, currentMonth);
      
      // Generate monthly data
      this.monthlyData = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'].map((month, index) => {
        // Generate random sales data
        const sales = index === currentMonth ? 500000 : Math.random() * 400000;
        return {
          month,
          sales,
          daysWithData: 30,
          orderCount: Math.round(sales / 5000) // Rough estimate for demo
        };
      });
      
      // Calculate statistics
      this.calculateStatistics();
      
      // Set loading to false
      this.loading = false;
      
      // Render the chart
      this.$nextTick(() => {
        setTimeout(() => {
          this.renderMonthlySalesChart();
        }, 300);
      });
      
      this.toast.success('Demo data loaded successfully');
    },
  }
};
</script>

<style scoped>
.app-container {
  padding: 20px;
  margin-left: 260px;
  transition: margin-left 0.3s;
}

.sales-container {
  max-width: 1400px;
  margin: 0 auto;
}

.header-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.section-title {
  font-size: 24px;
  color: #333;
  margin: 0;
}

.dashboard-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 20px;
  margin-bottom: 30px;
}

.card {
  background: white;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  padding: 20px;
}

.stats-card {
  text-align: center;
}

.stats-card h3 {
  font-size: 16px;
  color: #777;
  margin-bottom: 8px;
  font-weight: 500;
}

.stat-value {
  font-size: 26px;
  font-weight: 700;
  color: #333;
}

.charts-section {
  display: grid;
  grid-template-columns: 1fr;
  gap: 20px;
  margin-bottom: 30px;
}

.chart-card {
  min-height: 400px;
}

.chart-card h3 {
  font-size: 18px;
  color: #555;
  margin-bottom: 15px;
  font-weight: 500;
}

.chart-container {
  height: 350px;
  position: relative;
}

/* Responsive adjustments */
@media (max-width: 1024px) {
  .dashboard-row {
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  }
}

@media (max-width: 768px) {
  .app-container {
    margin-left: 0;
    padding: 15px;
  }
  
  .dashboard-row {
    grid-template-columns: 1fr;
  }
  
  .charts-section {
    grid-template-columns: 1fr;
  }
}

/* Add this new CSS for the top products table */
.top-products-section {
  margin-bottom: 30px;
}

.top-products-card {
  min-height: auto;
  padding: 20px;
  background: white;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}

.top-products-card h3 {
  font-size: 20px;
  color: #333;
  margin-top: 0;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid #eee;
  font-weight: 600;
}

.top-products-chart-container {
  padding: 10px 0;
}

.product-bars {
  display: flex;
  flex-direction: column;
  gap: 25px;
  padding: 10px 0;
}

.product-bar-item {
  display: flex;
  align-items: center;
  width: 100%;
  gap: 15px;
}

.product-rank {
  font-size: 16px;
  font-weight: bold;
  color: #666;
  min-width: 36px;
  text-align: center;
}

.product-image-container {
  width: 56px;
  height: 56px;
  border-radius: 5px;
  overflow: hidden;
  flex-shrink: 0;
  background-color: #f8f8f8;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px solid #eee;
}

.product-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.product-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.product-name-container {
  margin-bottom: 2px;
}

.product-name {
  font-size: 16px;
  font-weight: 500;
  color: #333;
  text-align: left;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 300px;
}

.bar-container {
  position: relative;
  height: 24px;
  background-color: #f0f0f0;
  border-radius: 5px;
  overflow: hidden;
}

.product-bar {
  height: 100%;
  transition: width 0.5s ease;
}

.product-sales {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 14px;
  font-weight: 600;
  color: #333;
  z-index: 2;
}

.product-details {
  display: flex;
  justify-content: space-between;
  font-size: 13px;
  color: #777;
  padding: 0 2px;
}

.no-data-message {
  text-align: center;
  color: #999;
  padding: 20px 0;
  font-style: italic;
}

.loading-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 30px 0;
}

.loading-spinner {
  border: 4px solid #f3f3f3;
  border-top: 4px solid #d84666;
  border-radius: 50%;
  width: 30px;
  height: 30px;
  animation: spin 1s linear infinite;
  margin-bottom: 10px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.data-source-info {
  text-align: right;
  padding: 8px 15px;
  color: #888;
  font-size: 0.8rem;
  border-top: 1px solid #eee;
  margin-top: 5px;
}

/* Add this new CSS for the predictions section */
.predictions-section {
  margin-bottom: 30px;
}

.predictions-card {
  min-height: auto;
  padding: 20px;
  background: white;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}

.predictions-card h3 {
  font-size: 20px;
  color: #333;
  margin-top: 0;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid #eee;
  font-weight: 600;
}

.prediction-summary {
  margin-bottom: 20px;
}

.prediction-products {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.prediction-product {
  display: flex;
  align-items: center;
  gap: 10px;
}

.prediction-rank {
  font-size: 16px;
  font-weight: bold;
  color: #666;
  min-width: 36px;
  text-align: center;
}

.prediction-name {
  font-size: 16px;
  font-weight: 500;
  color: #333;
  text-align: left;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 300px;
}

.prediction-value {
  font-size: 16px;
  font-weight: 700;
  color: #333;
}

.prediction-info {
  text-align: right;
  padding: 8px 15px;
  color: #888;
  font-size: 0.8rem;
  border-top: 1px solid #eee;
  margin-top: 5px;
}

/* Add this at the top of your existing styles */
.refresh-button {
  cursor: pointer;
  font-size: 14px;
  margin-left: 10px;
  color: #888;
  vertical-align: middle;
  transition: color 0.3s;
}

.refresh-button:hover {
  color: #2c7be5;
}
</style> 