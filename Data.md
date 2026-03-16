import axios from 'axios';
import React, { createContext, useState, useEffect } from 'react';

export const DataContext = createContext();

export const DataProvider = ({ children }) => {

    const [apiData, setApiData] = useState({
        MouData: [],
        HrComplainsData: [],
        TrainingData: [],
        AuditData: [],
        error: [],
        DashboardData: [],
        PoData: [],
        AllAuditData: [],
        HrTrainingData: [],
        EvaluateData: [],
        // InvoiceData: [],
        // PaymentsRecivedData: [],
        // CreditNotesData: [],
        StatementData: [],
        clientId: 0,
    });
    const [loading, setLoading] = useState(true);


    const getCurrentAndPreviousYearDates = () => {
        const today = new Date(); // Get the current date
        const currentYear = today.getFullYear(); // Extract the current year
        const currentDate = today.toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' }).replace(/ /g, '-'); // Format current date

        // Get the previous year and format it as 01-Jan-YYYY
        const previousYearDate = `01-Jan-${currentYear - 1}`;

        return {
            currentDate,
            previousYearDate
        };
    };

    const { currentDate, previousYearDate } = getCurrentAndPreviousYearDates();

    console.log('Current Date:', currentDate);
    console.log('Previous Year Date:', previousYearDate);
const fetchResgisterSfData = async (creatorClientID) => {
        console.log("creatorClientID : ", creatorClientID);
        try {
            let regSfResponse = [];
            let i = 1;
            while (true) {
                const regResponse = await ZOHO.CREATOR.API.getAllRecords({
                    appName: "ops",
                    reportName: "Register_Subform_Main_Report",
                    pageSize: 200,
                    page: i,
                    criteria: `Generated_By.ID==${creatorClientID}`,
                });
                console.log("regResponse : ", regResponse);
                regSfResponse.push(...regResponse.data);
                console.log('regResponse count', regResponse.data.length)
                if (regResponse.data.length < 200) {
                    break;
                }
                i += 1;
            }
            setApiData((prev) => ({ ...prev, RegisterSfData: regSfResponse }));
            console.log('RegisterSubformData response', regSfResponse)
        } catch (error) {
            console.error('Error fetching RegisterSubformData data:', error);
        }
    };
    
    const initializeAndFetchData = async () => {
        console.log("Fetching Data...")
        try {
            await ZOHO.CREATOR.init();
            const userDetails = ZOHO.CREATOR.UTIL.getInitParams();
            console.log('userDetails from Client View', userDetails);

            const clientResponse = await ZOHO.CREATOR.API.getAllRecords({
                appName: "ops",
                reportName: "Client_View",
                criteria: `UserId = "${userDetails.loginUser}"`,
            });
            console.log('client id', clientResponse);

            const CRMClientId = clientResponse.data[0].CRM_Client.ID;
            const creatorClientID = clientResponse.data[0].ZWP_Client.ID;
            const creatorClientName = clientResponse.data[0].ZWP_Client.display_value;
            console.log("creatorClient ID : ", creatorClientID);
            console.log("creatorClient Name : ", creatorClientName);
            // setApiData((prev) => ({ ...prev, clientId: creatorClientID }));
            setApiData((prev) => ({ ...prev, clientId: creatorClientID, clientName: creatorClientName }));

            fetchResgisterSfData(creatorClientID)

            // const clientResponse = await ZOHO.CREATOR.API.getAllRecords({
            //     appName: "ops",
            //     reportName: "Client_View",
            //     criteria: `Generated_By = "${generated_id.loginUser}"`,
            // });
            // console.log('client id', clientResponse);

            await fetchDashboardData(clientResponse.data[0].ZWP_Client.ID);
            setLoading(false);
        } catch (error) {
            console.error('Error in initialization:', error);
            setApiData((prev) => ({ ...prev, error: 'Failed to initialize or fetch client data' }));
            setLoading(false);
        }
    };

    useEffect(() => {
        initializeAndFetchData();
    }, []);

    return (
        // <DataContext.Provider value={{ apiData, loading, getStatementData }}>
        // <DataContext.Provider value={{ apiData, loading, fetchMouData, fetchHrData, fetchTrainingData, fetchAuditData, fetchHrTrainingData, fetchPoData, fetchDashboardData, getTemplatesOfInvoicePaymentCredit, getStatementData }}>
        <DataContext.Provider value={{ apiData, loading, getTemplatesOfInvoicePaymentCredit, getStatementData }}>
            {children}
        </DataContext.Provider>
    );
};
