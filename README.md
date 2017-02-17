# sash

package com.seleniumtests.browserfactory;

import java.net.MalformedURLException;
import java.net.URL;
import java.util.concurrent.TimeUnit;

import org.apache.http.HttpHost;
import org.apache.http.impl.client.DefaultHttpClient;
import org.apache.http.message.BasicHttpEntityEnclosingRequest;
import org.apache.http.util.EntityUtils;
import org.json.JSONObject;
import org.openqa.selenium.UnsupportedCommandException;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.remote.CapabilityType;
import org.openqa.selenium.remote.DesiredCapabilities;
import org.openqa.selenium.remote.RemoteWebDriver;

import com.seleniumtests.core.SeleniumTestsContextManager;
import com.seleniumtests.customexception.ConfigurationException;
import com.seleniumtests.driver.BrowserType;
import com.seleniumtests.driver.DriverConfig;
import com.seleniumtests.driver.screenshots.ScreenShotRemoteWebDriver;
import com.seleniumtests.reporter.TestLogging;
import com.seleniumtests.util.helper.WaitHelper;

public class SeleniumGridDriverFactory extends AbstractWebDriverFactory implements IWebDriverFactory {

    public Selenium(final DriverConfig cfg) {
        super(cfg);
    }

   
    public DesiredCapabilities createCapabilityByBrowser(DriverConfig webDriverConfig, DesiredCapabilities capabilities){

    	switch (webDriverConfig.getBrowser()) {

	        case FIREFOX :
	            capabilities.merge(new FirefoxCapabilitiesFactory().createCapabilities(webDriverConfig));
	            break;
	
	        case INTERNETEXPLORER :
	        	capabilities.merge(new IECapabilitiesFactory().createCapabilities(webDriverConfig));
	            break;
	
	        case CHROME :
	        	capabilities.merge(new ChromeCapabilitiesFactory().createCapabilities(webDriverConfig));
	            break;
	
	        default :
	            break;
	    }
    	
    	return capabilities;
    }
    
   
    private DesiredCapabilities createSpecificGridCapabilities(DriverConfig webDriverConfig) {
    	DesiredCapabilities capabilities = new DesiredCapabilities();
    	capabilities.setCapability(CapabilityType.PLATFORM, webDriverConfig.getPlatform().toLowerCase());
    	
    	if (SeleniumTestsContextManager.isMobileTest()) {
    		capabilities.setCapability(CapabilityType.VERSION, webDriverConfig.getMobilePlatformVersion());
    	}
    	
    	return capabilities;
    }
    
    @Override
    public WebDriver createWebDriver() {
        DriverConfig webDriverConfig = this.getWebDriverConfig();
        URL url;

        try {
			url = new URL(webDriverConfig.getHubUrl());
		} catch (MalformedURLException e1) {
			throw new ConfigurationException(String.format("Hub url '%s' is invalid: %s", webDriverConfig.getHubUrl(), e1.getMessage()));
		}

        DesiredCapabilities capabilities = createSpecificGridCapabilities(webDriverConfig);
        if (SeleniumTestsContextManager.isDesktopWebTest()) {
        	capabilities = createCapabilityByBrowser(webDriverConfig, capabilities);
        } else if (SeleniumTestsContextManager.isMobileTest()) {
        	if("android".equalsIgnoreCase(webDriverConfig.getPlatform())) {
        		capabilities = new AndroidCapabilitiesFactory(capabilities).createCapabilities(webDriverConfig);
	        } else if ("ios".equalsIgnoreCase(webDriverConfig.getPlatform())){
	        	capabilities = new IOsCapabilitiesFactory(capabilities).createCapabilities(webDriverConfig);
	        } else {
	        	throw new ConfigurationException(String.format("Platform %s is unknown for mobile tests", webDriverConfig.getPlatform()));
	        }
        } else {
        	throw new ConfigurationException("Remote driver is supported for mobile and desktop web tests");
        }
        

        if ((BrowserType.FIREFOX).equals(webDriverConfig.getBrowser())) {
            driver = getDriverFirefox(url, capabilities);
        } else {
            driver = new ScreenShotRemoteWebDriver(url, capabilities);
        }

        setImplicitWaitTimeout(webDriverConfig.getImplicitWaitTimeout());
        if (webDriverConfig.getPageLoadTimeout() >= 0) {
            setPageLoadTimeout(webDriverConfig.getPageLoadTimeout(), webDriverConfig.getBrowser());
        }

        this.setWebDriver(driver);

        runWebDriver(url);

        return driver;
    }
    
    private WebDriver getDriverFirefox(URL url, DesiredCapabilities capability){
    	driver = null;
    	try {
            driver = new ScreenShotRemoteWebDriver(url, capability);
        } catch (RuntimeException e) {
            if (e.getMessage().contains(
                        "Unable to connect to host localhost on port 7062 after 45000 ms. Firefox console output")) {
                TestLogging.log("Firefox Driver creation got port customexception, retry after 5 seconds");
                WaitHelper.waitForSeconds(5);
                driver = new ScreenShotRemoteWebDriver(url, capability);
            } else {
                throw e;
            }
        }
    	return driver;
    }
    
    private void runWebDriver(URL url){
    	String hub = url.getHost();
        int port = url.getPort();

        // logging node ip address:
        DefaultHttpClient client = new DefaultHttpClient();
        try {
            HttpHost host = new HttpHost(hub, port);
            
            String sessionUrl = "http://" + hub + ":" + port + "/grid/api/testsession?session=";
            URL session = new URL(sessionUrl + ((RemoteWebDriver) driver).getSessionId());
            BasicHttpEntityEnclosingRequest req;
            req = new BasicHttpEntityEnclosingRequest("POST", session.toExternalForm());

            org.apache.http.HttpResponse response = client.execute(host, req);
            String responseContent = EntityUtils.toString(response.getEntity());
            
            JSONObject object = new JSONObject(responseContent);
            String proxyId = (String) object.get("proxyId");
            String node = proxyId.split("//")[1].split(":")[0];
            String browserName = ((RemoteWebDriver) driver).getCapabilities().getBrowserName();
            String version = ((RemoteWebDriver) driver).getCapabilities().getVersion();
            TestLogging.info("WebDriver is running on node " + node + ", " + browserName + version + ", session "
                    + ((RemoteWebDriver) driver).getSessionId());
            
        } catch (Exception ex) {
        	logger.error(ex);
        } finally {
        	client.close();
        }
    }

	@Override
	protected WebDriver createNativeDriver() {
		return null;
	}
}
