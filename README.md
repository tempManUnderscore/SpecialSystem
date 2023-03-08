# SpecialSystem
THIS IS NOT WHAT YOU THINK IT IS

using OpenQA.Selenium.Chrome;
using OpenQA.Selenium;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Navigation;
using System.Windows.Shapes;
using System.Security.Policy;
using Selenium.WebDriver.WaitExtensions;
using System.Xml.Linq;
using HtmlAgilityPack;
using System.Net;
using System.Drawing;
using System.IO;

namespace FINALDummyThiccAirScraper
{
    /// <summary>
    /// Interaction logic for MainWindow.xaml
    /// </summary>
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
            List<List<string>> getPollutantData = AirQualityWebScraper();
            List<List<string>> getPollutantDataUpdated = GridDisplaySetup(getPollutantData);
            DataDisplay(getPollutantDataUpdated);



            //names of pollutants, added on he end of the 2d list so it needs to bseparated
            List<string> getPollutantNames = getPollutantData[getPollutantData.Count - 1];
            //remove this list from the 2d list
            getPollutantData.RemoveAt(getPollutantData.Count - 1);
        }

        public List<List<string>> AirQualityWebScraper()
        {

            //add the user's input to the url to be ultimately scraped from

            string url = "https://aqicn.org/city/south-africa/nkangala-dm/witbank-dea/";
            IWebDriver driver = new ChromeDriver();
            
            //wait for the page to load elements ==VITAL FOR PREVENTING ElementNotVisibleException==
            driver.Manage().Timeouts().ImplicitWait = TimeSpan.FromSeconds(10);

            driver.Navigate().GoToUrl(url);

            //get the table of data to be extracted
            var getRowIDValues = driver.FindElements(By.XPath("/html/body/div[10]/center/div[3]/div[1]/div/div/div/div/div/div/table[3]/tbody/tr"));

            //append the table ids for each value in a separate array
            List<string> tblIDValues = new List<string>();
            foreach (var element in getRowIDValues)
            {
                tblIDValues.Add(element.GetAttribute("id"));
            }

            //removes the empty value as its unnecessary
            tblIDValues.Remove("");

            //removes anything else other than the pollutants from the pollutantID list
            tblIDValues.Remove("tr_w");
            tblIDValues.Remove("tr_h");
            tblIDValues.Remove("tr_p");
            tblIDValues.Remove("tr_t");

            //new list is made for spliced values that will be used to identify other html elements at a later web scraping stage
            List<string> pollutantNamesSpliced = new List<string>();
            
            //iterates through each element to add pollutant values that are spliced - will be used later
            for (int valueIndex = 0; valueIndex < tblIDValues.Count; valueIndex++)
            {
                pollutantNamesSpliced.Add(tblIDValues[valueIndex].Substring(3));
            }

            //similar to above iterations, but these values will be used for the UI display, the front end of the application
            List<string> pollutantNameDisplay = new List<string>();
            for (int pollutantIndex = 0; pollutantIndex < tblIDValues.Count; pollutantIndex++)
            {
                //xpath of pollutant name
                string pollutantXpath = string.Format("/html/body/div[10]/center/div[3]/div[1]/div/div/div/div/div/div/table[3]/tbody/tr[{0}]/td[1]/div/div/span", pollutantIndex + 2);
                var getPollutantText = driver.FindElement(By.XPath(pollutantXpath)).Text;
                //only gets the test that we want
                pollutantNameDisplay.Add(getPollutantText.Substring(0,getPollutantText.Length-4));
            }

            //getting current, min, max values for each pollutent over 48 hours AND url of each pollutant's graph display

            //a 2d list for easier collation
            List<List<string>> combinedPollutantData = new List<List<string>>();

            for (int valueIndex = 0; valueIndex < tblIDValues.Count; valueIndex++)
            {
                List<string> specificPollutantData = new List<string>();

                //first we get all values
                string currentPollutantValuesXpath = string.Format("/html/body/div[10]/center/div[3]/div[1]/div/div/div/div/div/div/table[3]/tbody/tr[{0}]//*", valueIndex + 2);
                var getValues = driver.FindElements(By.XPath(currentPollutantValuesXpath));

                //then we gather all of the values' text values
                foreach (var element in getValues)
                {
                    //for each element, if the text value isnt empty
                    if (element.Text != "")
                    {   // and if the value can be parsed into an int value
                        if ((int.TryParse(element.Text, out int number)))
                        {
                            //add it to the data set
                            specificPollutantData.Add(element.Text);
                        }
                    } 
                }
                // data values: 0=current, 1=min, 2=max

                //now we also get src url of the pollutant graph that we can use in the air quality data dashboard
                string currentPollutantGraphXpath = string.Format("/html/body/div[10]/center/div[3]/div[1]/div/div/div/div/div/div/table[3]/tbody/tr[{0}]/td[3]/img", valueIndex + 2);
                var getGraphUrl = driver.FindElement(By.XPath(currentPollutantGraphXpath)).GetAttribute("src");
                specificPollutantData.Add(getGraphUrl);

                //append these data bits to the overall collection
                combinedPollutantData.Add(specificPollutantData);


            }
            //we can only return 1 value, so add the pollutant names to the end of the combined pollutant data list
            combinedPollutantData.Add(pollutantNameDisplay);

            //close driver
            driver.Close();
            return combinedPollutantData;

        }

        public List<List<string>> GridDisplaySetup(List<List<string>> pollutantDataValues)
        {

            //pollutant names are still on the end of this list (btw)
            //create the columns necessary for the data display.
            for (int i = 0; i < 5; i++)
            {
                ColumnDefinition gridCol = new ColumnDefinition();
                gridCol.Width = GridLength.Auto;
                gridAirData.ColumnDefinitions.Add(gridCol);
            }
            //create the top row of the grid, used for the 
            RowDefinition gridTopRow = new RowDefinition();
            gridAirData.RowDefinitions.Add(gridTopRow);

            for (int i = 0; i < 5; i++)
            {
                Label labelGridTitle = new Label();
                labelGridTitle.FontSize = 20;

                switch (i)
                {
                    case 0:
                        labelGridTitle.Content = "Pollutant";
                        Grid.SetColumn(labelGridTitle, i);      
                        Grid.SetRow(labelGridTitle, 0);
                        gridAirData.Children.Add(labelGridTitle);
                        break;
                    case 1:
                        labelGridTitle.Content = "Current";
                        Grid.SetColumn(labelGridTitle, i);
                        Grid.SetRow(labelGridTitle, 0);
                        gridAirData.Children.Add(labelGridTitle);
                        break;
                    case 2:
                        labelGridTitle.Content = "Past 48 Hours";
                        Grid.SetColumn(labelGridTitle, i);
                        Grid.SetRow(labelGridTitle, 0);
                        gridAirData.Children.Add(labelGridTitle);
                        break;
                    case 3:
                        labelGridTitle.Content = "Min";
                        Grid.SetColumn(labelGridTitle, i);
                        Grid.SetRow(labelGridTitle, 0);
                        gridAirData.Children.Add(labelGridTitle);
                        break;
                    case 4:
                        labelGridTitle.Content = "Max";
                        Grid.SetColumn(labelGridTitle, i);
                        Grid.SetRow(labelGridTitle, 0);
                        gridAirData.Children.Add(labelGridTitle);
                        break;

                }
            }

            //sees whether data is missing from any values
            
            for (int i = 0; i < pollutantDataValues.Count; i++)
            {
                if (pollutantDataValues[i].Count != 4)
                {//if data is missing, remove the pollutant as it isnt needed becuase data is incomplete
                    //accounts for the list of pollutants names at the end which we need
                    //if (!(pollutantDataValues.IndexOf(pollutant) == pollutantDataValues.Count-1))
                    //{
                    pollutantDataValues.Remove(pollutantDataValues[i]);
                    //}
                    //else
                    //{
                    //    break;
                    //}
                    
                }

            }

            //set the rows that are to be displayed based on pollutantDataValues count
            int rowsGenerated = pollutantDataValues.Count-1;
            for(int i = 0; i < rowsGenerated; i++)
            {
                RowDefinition gridRow = new RowDefinition();
                gridAirData.RowDefinitions.Add(gridRow);
            }

            return pollutantDataValues;
        }

        public void DataDisplay(List<List<string>> getPollutantValues)
        {
            List<string> getPollutantNames = getPollutantValues[getPollutantValues.Count - 1];
            //remove this list of pollutant names from the 2d list
            getPollutantValues.RemoveAt(getPollutantValues.Count - 1);

            for (int i = 0; i < getPollutantNames.Count; i++)
            {
                if (getPollutantNames[i].Length < 2)
                {   //removes any names that are not pollutants just incase any manages to slip though the process
                    getPollutantNames.Remove(getPollutantNames[i]);
                    //also removes assorted values
                    getPollutantValues.RemoveAt(i);
                }
            }
            for (int i = 0; i < getPollutantValues.Count; i++)
            {
                //adds another value so the pollutant name can get added to the grid display as its 1 more than the length of each data set
                getPollutantValues[i].Add("dummyValue");



                foreach (var pollutantDataBit in getPollutantValues[i])
                {
                    switch (getPollutantValues[i].IndexOf(pollutantDataBit))
                    {//ORDER OF GRAPH DISPLAY: NAME, CURRENT, IMG, MIN, MAX
                        case 0:
                            //this will be current value of pollutant (add its name on the end as its a selarate list)
                            Label pollutantStatistic1 = new Label();
                            pollutantStatistic1.FontSize = 20;
                            pollutantStatistic1.Name = "lbl"+i.ToString();
                            pollutantStatistic1.Content = pollutantDataBit;
                            Grid.SetRow(pollutantStatistic1, getPollutantValues.IndexOf(getPollutantValues[i]) + 1);
                            Grid.SetColumn(pollutantStatistic1, 1);
                            gridAirData.Children.Add(pollutantStatistic1);
                            break;
                        case 1: // min value of pollutant
                            Label pollutantStatistic2 = new Label();
                            pollutantStatistic2.Name = "lbl" + i.ToString();
                            pollutantStatistic2.Content = pollutantDataBit;
                            pollutantStatistic2.Foreground = Brushes.Green;
                            Grid.SetRow(pollutantStatistic2, getPollutantValues.IndexOf(getPollutantValues[i]) + 1);
                            Grid.SetColumn(pollutantStatistic2, 3);
                            gridAirData.Children.Add(pollutantStatistic2);
                            break;
                        case 2: // max value of pollutant
                            Label pollutantStatistic3 = new Label();
                            pollutantStatistic3.Name = "lbl" + i.ToString();
                            pollutantStatistic3.Content = pollutantDataBit;
                            pollutantStatistic3.Foreground = Brushes.Red;
                            Grid.SetRow(pollutantStatistic3, getPollutantValues.IndexOf(getPollutantValues[i]) + 1);
                            Grid.SetColumn(pollutantStatistic3, 4);
                            gridAirData.Children.Add(pollutantStatistic3);
                            break;
                        case 3:
                            Image pollutantImageGraph = new Image();
                            pollutantImageGraph.Name = "dataGraph"+i.ToString();
                            string getBase64value = pollutantDataBit.Substring(22);
                            byte[] binaryData = Convert.FromBase64String(getBase64value);
                            BitmapImage imgBitMap = new BitmapImage();
                            imgBitMap.BeginInit();
                            imgBitMap.StreamSource = new MemoryStream(binaryData);
                            imgBitMap.CacheOption = BitmapCacheOption.OnLoad;
                            imgBitMap.EndInit();
                            pollutantImageGraph.Source = imgBitMap;
                            Grid.SetRow(pollutantImageGraph, getPollutantValues.IndexOf(getPollutantValues[i]) + 1);
                            Grid.SetColumn(pollutantImageGraph, 2);
                            gridAirData.Children.Add(pollutantImageGraph);
                            break;
                        case 4:
                            Label pollutantStatistic = new Label();
                            pollutantStatistic.FontSize = 20;
                            pollutantStatistic.Name = "lbl" + i.ToString();
                            pollutantStatistic.Content = getPollutantNames[i];
                            Grid.SetRow(pollutantStatistic, getPollutantValues.IndexOf(getPollutantValues[i]) + 1);
                            Grid.SetColumn(pollutantStatistic, 0);
                            gridAirData.Children.Add(pollutantStatistic);
                            break;

                    }


                    //name of pollutant
                    
                }


            }

        }

    }
}
