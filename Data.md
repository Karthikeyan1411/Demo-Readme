import React, { createContext, useState, useEffect } from "react";

export const DataContext = createContext();

export const DataProvider = ({ children }) => {
  const [apiData, setApiData] = useState({ DashboardData: [], error: null });
  const [loading, setLoading] = useState(true);

  const fetchDashboardData = async () => {
    try {
      await ZOHO.CREATOR.init();

      let allRecords = [];
      let page = 1;
      const pageSize = 200;

      while (true) {
        const response = await ZOHO.CREATOR.API.getAllRecords({
          appName: "ops",
          reportName: "Client_View",
          criteria: `Generated_By == "Arliga EcoSpace"`,
          page,
          pageSize,
        });
        const batch = response.data || [];
        allRecords = [...allRecords, ...batch];
        if (batch.length < pageSize) break;
        page++;
      }

      console.log("Dashboard records fetched:", allRecords.length);
      setApiData({ DashboardData: allRecords, error: null });
    } catch (error) {
      console.error("Error fetching dashboard data:", error);
      setApiData({ DashboardData: [], error: "Failed to fetch dashboard data" });
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchDashboardData();
  }, []);

  return (
    <DataContext.Provider value={{ apiData, loading }}>
      {children}
    </DataContext.Provider>
  );
};
